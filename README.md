# About
A baby monitor (video only). Based on RPi Cam Control project.

## Features
Based on on a Raspberry Pi Zero W, this runs a little web server that anyone in the WLAN can connect to.
The Camera is infrared-capable, and the web UI offers a switch to enable some infrared lights to illuminate the scene.

This comes with a casing that fits to the "Babyhimmel" over our daughter's child bed.

In the end you can monitor your little one from any smartphone or computer (anything running a browser). It'll look like this:
![Web UI preview](https://user-images.githubusercontent.com/71769938/99910862-c1323b80-2cf0-11eb-94f2-f7fca728151c.png)


# Hardware
The components are pretty standard:
- A Raspberry Pi Zero W plus a suitable SD card
- A Camera for the Pi (tested with th standard one)
- An extra break-out-board supplying and holding the IR LEDs

TODO: schematic system, schematic IR board.

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
