# Wireless Broadband Communication System
The wireless broadband communication systems allows to send MavLink messages between the vehicle and ground station using a GSM connection. It can offer unlimited range but is limited by the supported area of the service provider. The cell towers are usually optimized to provide a good connection close to the ground. This is why the connection speed and stability can deteriorate at higher altitudes above ground.

The system uses an onboard computer, connected to the pixhawk with a FTDI cable, forwarding the messages to the relay server using MAVProxy. The internet connection on the onboard computer is provided with a LTE stick.

## Vehicle Setup

#### Companion Computer
The companion computer can be connected to the Pixhawk using a FTDI cable. Refer to [Companion Computer for Pixhawk Series](https://dev.px4.io/master/en/companion_computer/pixhawk_companion.html)  page in the dev guide for specific instruction on how to connect the companion computer to the pixhawk. Any computer able to run Ubuntu and providing two USB ports can be used, e.g. [Rasberry Pi 4](https://www.raspberrypi.org/products/) or [Intel Up Board](https://up-board.org/). In the following it is assumed that the onboard computer is already setup with Ubuntu and wired to the Pixhawk.

##### LTE Stick
Depending on the Linux distribution and version the LTE stick (e.g. Huawei E3372h) might work just out of the box. Chances are though that you will need to do some tweaking to make everything work smoothly.

When you plug the LTE stick, ideally it is configured automatically as a network device and assigned an IP. Check your network interfaces using e.g. `ifconfig -a` or `ip address`. The list should contain a device for the LTE stick, the IP address depends on the model of the stick). If not, continue reading.

As a next step, you can check if the USB device has been recognized and in what mode it is:

`lsusb`

Observe the output to check if the LTE stick is enabled as an USB media device, a so-called mass storage device (e.g. `ID 12d1:1f01`in the output in case of an Huawei stick). This mode is enabled in order to provide (Windows) users with some software. To enable the stick as a network adapter on Linux systems, a software called `usb_modeswitch` can be used. On Ubuntu based systems, you can install it easily using apt:

`sudo apt-get install usb-modeswitch`

Now you can test, if you can bring up the device as a network interface manually:

`sudo usb_modeswitch -v 12d1 -p 1f01 -J`

Now, the LTE stick should be enabled as a network interface. `lsusb` should now give a line containing `ID 12d1:14dc`. If not, try the `usb_modeswitch` command once more.

You should now also have the network interface listed when running `ifconfig -a`. Depending on your Linux distribution, the name of that network interface can vary, maybe depending on which LTE stick you're using.

To have the LTE stick enabled as a network interface every time it is plugged, you might need to set up the following udev rule, for instance in /etc/udev/rules.d/11-custom-usb_modeswitch.rules

```
# LTE USB stick
SUBSYSTEM=="usb", ACTION=="add|change", ATTRS{idVendor}=="12d1", ATTRS{idProduct}=="1f01", RUN+="/usr/sbin/usb_modeswitch -v%s{idVendor} -p%s{idProduct} -J"
```

If you prefer a fixed network interface name, you might need to set this up. Depending on your Linux version, there are several ways to give a specific name to the network interface created by the LTE stick, such as:

    - [creating a systemd link in /etc/systemd/network](https://www.freedesktop.org/software/systemd/man/systemd.link.html)
    - creating a udev rule

On Ubuntu 16.04, the following udev rule `/etc/udev/rules.d/10-rename-network-devs.rules` names the LTE stick network interface lte_X_, i.e. lte0 if you only plug one LTE stick:

`SUBSYSTEM=="net", ACTION=="add", ATTRS{idProduct}=="14dc", ATTRS{idVendor}=="12d1", NAME="lte%n"`

You then should be able to set up the interface by adding the following section to /etc/network/interfaces (IF still working on your linux system -- recent Ubuntu systems use [Netplan](https://ubuntu.com/server/docs) by default):

```
allow-hotplug lte0
iface lte0 inet dhcp
```

##### MAVProxy
MAVProxy forwards the message from the autopilot to the relay server and vice versa. Follow the [install guide](https://ardupilot.org/mavproxy/docs/getting_started/download_and_installation.html) to setup MAVProxy on the companion computer.


After successfully installing MAVProxy it can be run using the following command:

```
mavproxy.py --master=DEVICE,BAUDRATE --out=udpout:IP_ADDRESS:PORT
```

The `DEVICE` and the `BAUDRATE` have to be set to the port serial device and the baud rate of the connection to the pixhawk. The `IP_ADDRESS` and the `PORT` have to be set to the ip address of the relay server and the port where the relay script is configured to listen for messages.

By adding a second out the data can be streamed to multiple relay servers/ground stations, e.g.:` --out=IP_ADDRESS_2:PORT_2`

For more detailed instruction for MAVProxy refer to the respective [user guide](https://ardupilot.org/mavproxy/).

To automatically start MAVProxy in a screen first create the a file with the following content on the companion computer:

```
deflogin on
autodetach on

escape ^oo

caption always

screen 0
title "mavproxy"

stuff "mavproxy.py --master=DEVICE,BAUDRATE --out=udpout:IP_ADDRESS:PORT \015"
```

Then add the following line to `/etc/rc.local`:

```
runuser -l USER -c '/usr/bin/screen -dm -S lte -c PATH_TO_FILE'
```

Where `USER` reflects the user name in Ubuntu and the `PATH_TO_FILE` is the path to the previously created file.

#### Relay Server
Please refer to the [Long Range Communication](../advanced_features/long_range_com.md) chapter.

#### PX4 Autopilot
Any version of the PX4 version will work together with the wireless broadband communication link. Refer to the [companion computer setup page](https://dev.px4.io/master/en/companion_computer/pixhawk_companion.html) for instructions on setting up the MavLink stream. Using the provided PX4 parameter only previously defined modes can be used, e.g. onboard or normal. If a custom configuration is desired it can be achieved by adding a custom startup script for the respective MavLink instance. In our experience the bandwith of the connection allows for higher rates than the normal mode, e.g. onboard mode.

## Descriptions and Limitations of the WBC system
The first message sent from the vehicle to the ground station, e.g. a heartbeat, will initialize the link on the relay server and in QGroundControl as well. The IP address and port of the onboard computer from the plane is logged on the relay server for any received message enabling a dynamic IP address on the onboard computer. After a certain time period without any new message from the vehicle resets the IP address and port. This timeout value can be adjusted in the `relay.cfg` file. In case the IP address of the relay server is dynamic as well the correct IP address has to be set when starting MAVProxy every time at vehicle boot.

The WBC link operates similar to any regular low latency telemetry link. It supports all operations of a low latency link, e.g. virtual joystick, up- and downloading missions, or changing parameter values. The latency of the link depends on the quality of the internet connection of the onboard computer but is in general quite low. During test flights controlling the plane with a virtual joystick over the WBC link was evaluated and no significant delay was noticed. The WBC link classifies as a low latency link and in case of multiple links it will be preferred over any high latency link, e.g. SatCom link.

The current implementation of the relay server only supports a single active vehicle at any time. Connecting multiple vehicles over a wireless broadband link at the same time would either require multiple relay servers, one for each vehicle, or upgrading the software of the relay server to support multiple vehicle on one server.

## Running the System
1. Start *QGroundControl* and connect the WBC link.

2. Open a terminal on the ground station computer and change to the location of the *SatComInfrastructure* repository. 
   Then start the `udp2mqtt.py` script:
   ```
   ./udp2mqtt.py
   ```
3. Power up the vehicle.

4. Wait until the first heartbeat message is received on QGC and the connection to the vehicle is established. The WBC link is now ready to use.

## Troubleshooting
* No messages are transmitted
  * Check the following subsystems verify that
    * the internet connection of the onboard computer is stable.
    * MAVProxy is started and the correct IP address and port of the relay server are set.
    * the respective ports on the relay server are open and that the relay script is running.
    * the udp2mqtt script is running and configured correctly.
    * the WBC link is connected in QGC and the ground station computer has a stable internet connection
  * On the relay server the transmission rate is printed for every 1000th received message from the vehicle. If such a message in the terminal is observed it indicates that the connection from the vehicle to the relay server is working.
  * By changing the logging level in the [udp2mqtt script](https://github.com/acfloria/SatComInfrastructure/blob/master/udp2mqtt.py#L186) to `INFO` every received message from the relay server on the ground station computer is visualized by a message in the terminal.

