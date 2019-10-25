# Raspberry Pi /w piCore (TinyCoreLinux) Wifi Config

A Node application which makes connecting your Raspberry Pi running piCore to your home WiFi easier.

Tested on Raspberry Pi 3B with piCore v9.0.3

## RPi Compatibility

This has only been tested on a RPi 3B and will soon be tested on an RPi Zero W with piCore v9.0.3. I have no plans to test it on other combinations but feel free to try it out and contribute fixes to any potential issues.

## Motivation

When unable to connect to a WiFi network, this service will turn the RPi into a Wireless AP. This allows us to connect to it via a phone or other device and configure an existing WiFi network.

Once configured, it prompts the Pi to restart networking with the appropriate WiFi credentials. If this process fails, it immediately re-enables the Pi as an AP which can be configurable again.

This project broadly follows these [instructions](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md) in setting up a Raspberry Pi as a Wireless AP.

## Requirements

The NodeJS modules required are pretty much just `underscore`, `async`, `move-file` and `express`.

The web application requires `angular` and `font-awesome` to render correctly. To make the deployment of this easy, one of the other requirements is `bower`.

If you do not have `bower` installed already, you can install it globally by running: `sudo npm install bower -g` or use the `npm script`: `npm run prep`.

## Install

- Download piCore: http://tinycorelinux.net/9.x/armv6/releases/RPi/piCore-9.0.3.zip

- Install piCore: https://iotbytes.wordpress.com/picore-tiny-core-linux-on-raspberry-pi/

- Get WiFi working: https://iotbytes.wordpress.com/make-raspberry-pi-3-built-in-wifi-module-work-with-picore/

- Install the required dependencies: `tce-load -wi dhcpcd.tcz hostapd.tcz dnsmasq.tcz iw.tcz node.tcz`

- Install the optional dependencies: `tce-load -wi nano.tcz git.tcz`

- Run the app, this will do a number of things including using `nvm` to upgrade to node v8.16.2, installing `bower` and running an `install`, running an `npm update` and disconnecting from any current WiFi network:

```sh
git clone https://github.com/alanmosely/rpi-picore-wifi-conf.git ; \
cd rpi-picore-wifi-conf ; \
sudo npm run full-setup ; \
source ../.profile ; \ # updates to the upgraded node
sudo npm start
```

## Setup the app as a service

There is a startup script included to make service management easier. Do remember that the application is assumed to be installed under `/home/tc/rpi-picore-wifi-conf`. Feel free to change this in the `assets/init.d/rpi-picore-wifi-conf` file.

```sh
$sudo cp assets/init.d/rpi-picore-wifi-conf /etc/init.d/services/rpi-picore-wifi-conf
$sudo chmod +x /etc/init.d/rpi-picore-wifi-conf  
$sudo echo "/etc/init.d/services/rpi-picore-wifi-conf start" >> bootlocal.sh
```

### Gotchas

#### `hostapd`

The `hostapd` application does not like to behave itself on some WiFi adapters (RTL8192CU et al). This link does a good job explaining the issue and the remedy: [Edimax Wifi Issues](http://willhaley.com/blog/raspberry-pi-hotspot-ew7811un-rtl8188cus/). The gist of what you need to do is as follows:

```sh
# run iw to detect if you have a rtl871xdrv or nl80211 driver
$iw list
```

If the above says `nl80211 not found.` it means you are running the `rtl871xdrv` driver and probably need to update the `hostapd` binary as follows:

```sh
$cd raspberry-wifi-conf
$sudo mv /usr/sbin/hostapd /usr/sbin/hostapd.OLD
$sudo mv assets/bin/hostapd.rtl871xdrv /usr/sbin/hostapd
$sudo chmod 755 /usr/sbin/hostapd
```

Note that the `wifi_driver_type` config variable is defaulted to the `nl80211` driver. However, if `iw list` fails on the app startup, it will automatically set the driver type of `rtl871xdrv`. Remember that even though you do not need to update the config / default value - you will need to use the updated `hostapd` binary bundled with this app.

## Usage

This is approximately what occurs when we run this app:

1. Check to see if we are connected to a WiFi AP
2. If connected to a WiFi, do nothing -> exit
3. (if not WiFi, then) Convert RPI to act as an AP (with a configurable SSID)
4. Host a lightweight HTTP server which allows for the user to connect and configure the RPi's WiFi connection. The interfaces exposed are RESTy so other applications can similarly implement their own UIs around the data returned
5. Once the RPI is successfully configured, reset it to act as a WiFi device (not AP anymore), and setup its WiFi network based on what the user selected
6. At this stage, the RPI is named, and has a valid WiFi connection which it is now bound to

## User Interface

In my config file, I have set up the static ip for my PI when in AP mode to `192.168.44.1` and the AP's broadcast SSID to `rpi-config-ap`. These are images captured from my osx dev box.

Step 1: Power on Pi which runs this app on startup (assume it is not configured for a WiFi connection). Once it boots up, you will see `rpi-config-ap` among the WiFi connections.  The password is configured in config.json.

<img src="https://raw.githubusercontent.com/sabhiram/public-images/master/raspberry-wifi-conf/wifi_options.png" width="200px" height="160px" />

Step 2: Join the above network, and navigate to the static IP and port we set in config.json (`http://192.168.44.1:88`), you will see:

<img src="https://raw.githubusercontent.com/sabhiram/public-images/master/raspberry-wifi-conf/ui.png" width="404px" height="222px" />

Step 3: Select your home (or whatever) network, punch in the WiFi passcode if any, and click `Submit`. You are done! Your Pi is now on your home WiFi!!
