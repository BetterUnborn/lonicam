# About
This project features a privacy-friendly baby monitor (video only) for the enthusiast. 
- Basis for it is a Raspberry Pi Zero with a camera
- Embedded within a custom-created case
- Only sends data in local network
- Accessible with any device (only web browser is required)
- Manual control of IR LEDs to illuminate picture in dark conditions

We use this thing on a day-to-day base, it is mounted above our daughter's bed. If the power supply and cable is good, it runs very stable for weeks.
![Web UI preview](https://user-images.githubusercontent.com/71769938/99910862-c1323b80-2cf0-11eb-94f2-f7fca728151c.png)


# Hardware
The components are pretty standard:
- A Raspberry Pi Zero W plus a suitable SD card
- A Camera for the Pi (tested with th standard one)
- An extra break-out-board supplying and holding the IR LEDs

## The IR board
The schematic can be seen below:
![image](https://user-images.githubusercontent.com/71769938/100269463-dd371680-2f56-11eb-8e9c-1adbab365089.png)
The Pi's GPIOs don't provide enough current to drive the LEDs, so a ULN2003 is used to amplify.

You can craft a PCB on your own, I just soldered it to a `Lochrasterplatine`.

# Casing
The files are uploaded as FreeCAD model and the STL export. It looks like this:
![LoniCAM_V3](https://user-images.githubusercontent.com/71769938/110521045-91783980-810f-11eb-8968-ea801ae30655.png)
The rectangular hole is the outlet for the 4 IR LEDs, the round hole is for the camera.

This casing is custom-created for her bed, you may need to modify it to properly mount it. You're welcome to reuse parts of the freecad design.

# Software setup

The software is based on '''Raspberry Pi OS''', so get it from https://www.raspberrypi.org/software/.

## Preparing SD card

- Write the Raspberry Pi OS image to the SD card, whichever way you prefer (I use `dd`)
- Create an empty file named `ssh` on the `boot` partition.
- Create a file `wpa_supplicant.conf` on the boot partition, with the following content:
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=DE

network={
    ssid="<your WLAN's SSID>"
    psk="<the WLAN's password>"
    key_mgmt=WPA-PSK
}

```
The "network" section can be duplicated to be capable to connect in multiple places.

## Booting up and basic setup

Now insert this SD card into the Pi and boot it up. Follow these steps after boot.
- Connect to the Pi via ssh: `ssh pi@raspberrypi` (maybe you need to look up the Pi's IP address in your router ...)
  - password is `raspberry`
- Edit `/etc/hostname` to contain whatever name you want to reach your cam at (for me this is `lonicam`)
- Run `sudo raspi-config`
  - Set your personal password
  - Enable camera interface
  - Extend primary partition to whole SD card
  - Set the correct timezone
- Reboot, reconnect ssh
- Run `apt update` and `apt dist-upgrade`
- Reboot, reconnect ssh
- Run `apt install vim wiringpi git`

## Installing camera service

Now download the RPi Cam Control project with `git clone https://github.com/silvanmelchior/RPi_Cam_Web_Interface.git LoniCAM`. After that cd into the `LoniCAM` folder. Run `sudo ./install.sh`. The web server is best case `lighttpd` (low on resources, and fully sufficient).

The installation process will take quite some time. Allow it to start the camera and then you will already be able to connect to the service.

## IR LED control

The IR LEDs are connected to the Pi's GPIO_12 pin, which is PWM capable. Thus the LED brightness can be controlled.

### Raspberry Pi OS

The PWM can be controlled directly via `sysfs` interface. Therefore PWM clock sources must be enabled in the overlay, so add this line to the `/boot/config.txt`:
```
dtoverlay=pwm
```

To enable extra buttons in the web UI to control the LED, create the script `/var/www/html/macros/ledsOn.sh` with this:
```
#!/bin/bash

# create the pwm control structures, if needed
if [ ! -e /sys/class/pwm/pwmchip0/pwm0 ]
then
	echo 0 > /sys/class/pwm/pwmchip0/export
fi
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period
echo 500000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle
echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable
```
The ratio of the values `500000` / `1000000` defines the duty cycle of the signal and hence the brightness, so it is set to 50% with this config.

Create the script `cat /var/www/html/macros/ledsOff.sh` with this content:
```
#!/bin/bash

if [ -e /sys/class/pwm/pwmchip0/pwm0/enable ]
then
	echo 0 > /sys/class/pwm/pwmchip0/pwm0/enable
fi
```

Set both scripts executable.


### Raspberry Pi OS before Bullseye

The package `wiringpi` got deprecated in the newest version of Raspberry Pi OS (which is based on Bullseye), hence this will only work in older version until Buster.

To enable extra buttons in the web UI to control the LED, create the script `/var/www/html/macros/ledsOn.sh` with this:
```
#!/bin/bash
gpio -g mode 12 pwm
gpio -g pwm 12 350
```
The value of `350` defines the brightness. `1024` means duty cycle of 100%. Adjust to your personal needs.

Create the script `cat /var/www/html/macros/ledsOff.sh` with this content:
```
#!/bin/bash
gpio -g mode 12 pwm
gpio -g pwm 12 0
```

Set both scripts executable.

### Configuration of server

Create the file `/var/www/html/userbuttons` with this content:
```
LICHT AN,ledsOn.sh,btn btn-success btn-lg,style="width:50%" autofocus
LICHT AUS,ledsOff.sh,btn btn-danger btn-lg,style="width:50%"
```

The next restart of the PI (or the web server) you will be greeted with 2 extra buttons to control the IR LEDs.

## Disable the Pi's green LED

Otherwise it is rather disturbing for the child. Add this line to `/etc/rc.local` file:

`echo kbd-altgrlock > /sys/class/leds/led0/trigger`

Since no keyboard is attached, this will do the trick, and the LED is permanently off. Oddly enough `none` did not work as expected.

## Auto-disable the IR LEDs based on timer

_This is a lazy workaround. Better it would be to program a timer to automatically disable after 60 seconds or so._
Run `crontab -e` and add this line:
```
*/5 * * * * (gpio -g pwm 12 0)
```
This will disable the IR LEDs automatically every 5 minutes. Just to ensure they don't permanently lit up.

## Enable the Pi's Watchdog

Basically it is beneficial to set up the Pi's watchdog ... sometimes it occured to me that it got stuck or hung (maybe due to power struggles). The watchdog will bring it back to life again.

Install the watchdog with `sudo apt install watchdog`. Edit the file `/etc/watchdog.conf` and enable the following lines:
```
watchdog-device        = /dev/watchdog
max-load-1             = 24
```

## Get Rid of Recording/Timelapsing Buttons

Find the `div` by the id `main-buttons` in the file `/var/www/html/index.php` and comment the whole element out (all 7 rows). It reduces the GUI so you don't see the unused and confusing buttons no more.

## Check SD card

Especially when shut down by just disconnecting the power, the SD card and the filesystems can be corrupted. This checks for bad blocks:
```
> sudo badblocks -n -v -s /dev/mmcblk0
```
This scans the file system:
```
> sudo e2fsck -f /dev/mmcblk0p2
```

