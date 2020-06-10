# Iridium/RockBlock Satellite Communication System

The satellite communication system (SatCom) can be used to provide long range high latency link between a ground station and a vehicle.

This topic describes how to set up and run a system that uses RockBlock as the service provider for the Iridium SBD Satellite Communication System. 
Given good signal quality, users can expect a median latency between 10 to 15 seconds. The latencies experienced while developing the framework are ranging from
5 seconds up to 180 seconds and were highly dependent on the flight location. Usually the connection was better in flight because the line of sight to the satellites was less likely to be blocked by terrain.

An unofficial status about the iridium system can be found [here](http://www.rod.sladen.org.uk/iridium.htm).

## Costs

The link running cost consists of a line rental and per message cost ([RockBlock Prices](https://docs.rockblock.rock7.com/docs/iridium-contract-costs):
* Each module must be activated by paying a monthly line rental fee
* Each message transmitted over the system costs one *credit* per 50 bytes. Bundles of credits can be bought from RockBlock. The price per credit depends on the bundle size.

Refer to the [RockBlock Documentation](https://docs.rockblock.rock7.com/docs) for a detailed explanation of the modules, running costs and *RockBlock* in general.

## Vehicle Setup

### Wiring

Connect the RockBlock module to a serial port of the Pixhawk. 
Due to the power requirements of the module it can only be powered over a high-power serial port as a maximum of 0.5 A at 5 V are required. 
If none is available/free then another power source which has the same ground level as the Pixhawk and can provide required power has to be setup. 
The details of the [connectors](https://docs.rockblock.rock7.com/docs/connectors) and the [power requirements](https://docs.rockblock.rock7.com/docs/power-supply) can be found in the RockBlock documentation.

TODO: wiring image

### Module

The module can either use the internal antenna or an external one connected to the SMA connector. 
To [switch between the two antennas modes](https://docs.rockblock.rock7.com/docs/switching-rockblock-9603-antenna-mode) the position of a small RF link cable needs to changed. 
If an external antenna is used always make sure that the antenna is connected to the module before powering it up to avoid damage to the module.

The default baud rate of the module is 19200. However, the PX4 *iridiumsbd* driver requires a baud rate of 115200 so it needs to be changed using the [AT commands](http://www.rock7mobile.com/downloads/IRDM_ISU_ATCommandReferenceMAN0009_Rev2.0_ATCOMM_Oct2012.pdf).

1. Connect to the module with using a 19200/8-N-1 setting and check if the communication is working using the command: `AT`. 
   The response should be: `OK`.
2. Change the baud rate:
   ```
   AT+IPR=9
   ```
3. Reconnect to the model now with a 115200/8-N-1 setting and save the configuration using:
   ```
   AT&W0
   ```

The module is now ready to be used with PX4.

### Software

[Configure the serial port](../peripherals/serial_configuration.md) on which the RockBlock module will run using [ISBD_CONFIG](../advanced_config/parameter_reference.md#ISBD_CONFIG).
There is no need to set the baud rate for the port, as this is configured by the driver.

> **Note** If the configuration parameter is not available in *QGroundControl* then you may need to [add the driver to the firmware](../peripherals/serial_configuration.md#parameter_not_in_firmware):
  ```
  drivers/telemetry/iridiumsbd
  ```

If you add the driver to a custom startup script please have a look at the [official startup command](https://github.com/PX4/Firmware/blob/master/src/drivers/telemetry/iridiumsbd/module.yaml) since it is important to give the driver time to power up and check if the driver did start correctly. 


## RockBlock Setup

When buying the first module on RockBlock an user account needs to be created in a first step.

Log in to the [account](https://rockblock.rock7.com/Operations) and register the RockBlock module under the `My RockBLOCKs`. 
Activate the line rental for the module and make sure that enough credits for the expected flight duration are available on the account.
When using the default settings one message per minute is sent from the vehicle to the ground station.

Set up a delivery group for the message relay server and add the module to that delivery group:

![Delivery Groups](../../assets/long_range_com/deliverygroup.png)

## Relay Server and Ground Station Computer Setup
Please refer to the [Long Range Communication](advanced_features/long_range_com.md) chapter.


## Setup Verification
It is assumed that the relay server is set up and operational according to the procedure explained in [Long Range Communication](advanced_features/long_range_com.md) chapter.

### Relay Server and Ground Station Computer Setup Verification

1. Open a terminal on the ground station computer and change to the location of the *SatComInfrastructure* repository. 
   Then start the **udp2mqtt.py** script:
   ```
   ./udp2mqtt.py
   ```
   If you see the following line `Connected with result code 0` it means that the connection to the relay server was successful.

2. Send a test message from your [RockBlock Account](https://rockblock.rock7.com/Operations) to the created delivery group in the `Test Delivery Groups` tab.

   If in the terminal where the `udp2mqtt.py` script is running within a couple of seconds the acknowledge for a message can be observed (`MQTT received message from telem/SatCom_from_plane`), then the RockBlock delivery group, the relay server and the udp2rabbit script are set up correctly:

![udp2rabbit message acknowledge](../../assets/long_range_com/verification.png)


### Full System Setup Verification
1. Start *QGroundControl* and connect only the high latency link.
2. Open a terminal on the ground station computer and change to the location of the *SatComInfrastructure* repository. 
   Then start the **udp2mqtt.py** script:
   ```
   ./udp2mqtt.py
   ```
3. Connect to the [system console](https://dev.px4.io/v1.9.0/en/debug/system_console.html) of the PX4 and then power up the drone and observe the output of the system console.
4. If one can observe `INFO  [iridiumsbd] starting` without any iridiumsb related warning it indicates that the driver did boot up properly.
5. If one can observe something similar to `INFO  [mavlink] mode: Iridium, data rate: 1200 B/s on /dev/iridium @ 115200B` it also indicates that the mavlink instance did start correctly.
6. In the system console check the driver status using `iridiumsb status`. The system will send an initial message to initialize the link which can be seen in the status either by `INFO  [iridiumsbd] TX session pending: 1` (message not sent yet) or nonzero `Last Heartbeat` (message already sent).
7. One could trace the message by checking the outputs in the [RockBlock user account](https://rockblock.rock7.com/Operations), on the relay server, and in terminal where the `./udp2mqtt.py` script is running. Once the message is received by *QGroundControl* it should indicate that the connection to a new vehicle was established. This verifies that all parts of the link are working.

## Descriptions and Limitations of the SatCom link
Due to the high latency and limited bandwidth of the SatCom link it does not support the same set of functionalities as a standart low latency link.

One such example is that in general only messages over the SatCom link are sent if the vehicle is armed. The idea is to limit the number of transmitted messages when the plane and if the plane is not armed on the ground it is anyway recommended to use a low latency link to setup/observe the vehicle status.

TODO
different timeouts (comm and commands sent, QGC and vehicle)
HL2 is accepted as a heartbeat
from the vehicle side by default only

Known issue: messages could get out of order (QGC side handled, vehicle side(commands) not as this would require significant mavlink changes)

## Running the System
This specific startup order makes sure that the initial message, which is used to initialize the link in QGC, is not missed. Any other order might still work but there is a small risk that the message might be missed (for example `./udp2mqtt.py` just forwards the message but does not check if it is received by QGC, so if QGC was not open the message is lost).

1. Start *QGroundControl*. Manually connect the high latency link first, then the regular telemetry link:

   ![Connect the High Latency link](../../assets/long_range_com/linkconnect.png)

2. Open a terminal on the ground station computer and change to the location of the *SatComInfrastructure* repository. 
   Then start the **udp2mqtt.py** script:
   ```
   ./udp2mqtt.py
   ```
3. Power up the vehicle.

4. Wait until the first `HIGH_LATENCY2` message is received on QGC.
   This can be checked either using the *MAVLink Inspector* widget or on the toolbar with the *LinkIndicator*. 
   If more than one link is connected to the active vehicle the *LinkIndicator* shows all of them by clicking on the name of the shown link:

   ![Link Toolbar](../../assets/long_range_com/linkindicator.jpg)

   The link indicator always shows the name of the priority link. Messages can be received over any link but commands are only sent from QGC to the vehicle through the priority link.

5. The satellite communication system is now ready to use. 
   The priority link is determined the following ways:
   * If no link is commanded by the user a regular low latency telemetry link is preferred over the high latency link.
   * The autopilot and QGC will fall back from the regular radio telemetry to the high latency link if the vehicle is armed and the radio telemetry link is lost (no MAVLink messages received for a certain time). 
   As soon as the radio telemetry link is regained QGC and the autopilot will switch back to it.
   * The user can select a priority link over the `LinkIndicator` on the toolbar. 
     This link is kept as the priority link as long as this link is active or the user selects another priority link:

     ![Prioritylink Selection](../../assets/long_range_com/linkselection.png)

## Troubleshooting

* Satellite communication messages from the airplane are received but no commands can be transmitted (the vehicle does not react)
  * Check the settings of the relay server and make sure that they are correct, especially the IMEI.
* No satellite communication messages from the airplane arrive on the ground station:
  * Check using the system console if the *iridiumsbd* driver started and if it did that a signal from any satellite is received by the module:
    ```
    iridiumsbd status
    ```
  * Make sure using the verification steps from above that the relay server, the delivery group and the `udp2rabbit.py` script are set up correctly.
  * Check if the link is connected and that its settings are correct.
  
* The IridiumSBD driver does not start:
  * Reboot the vehicle. 
    If that helps increase the sleep time in the `extras.txt` before the driver is started. 
    If that does not help make sure that the Pixhawk and the module have the same ground level. Confirm also that the baudrate of the module is set to 115200.

* The iridium mavlink instance does not start properly
  * Check the firmware version. Between v1.8 and v1.11 a bug in the mavlink module that caused the crash of the module on start up.

* A first message is received on the ground but as soon as the vehicle is flying no message can be transmitted or the latency is significantly larger (in the order of minutes)
  * Check the signal quality after the flight. 
    If it is decreasing during the flight and you are using the internal antenna consider using an external antenna. 
    If you are already using the external antenna try moving the antenna as far away as possible from any electronics or anything which might disturb the signal.
    Also make sure that the antenna is is not damaged.

* The first message is not transmitted although the pipeline was working earlier
  * Check if there is any satellite visible at the moment from your current location. [Satflare](http://www.satflare.com/track.asp?q=Iridium#MAP) offers on their webpage a tool to visualize the satellites visible on a given location on the earth. Just set your current location using the `Set your Location` button and check if the currently visible satellite is behind the terrain.


