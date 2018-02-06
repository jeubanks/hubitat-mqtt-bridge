# Hubitat Elevation MQTT Bridge
***System to share and control Hubitat Elevation device states in MQTT.***

Original Author:
https://hub.docker.com/r/stjohnjohnson/hubitat-mqtt-bridge/

I highly recommend looking at the original github above as it may have more information
that I have left out, or removed as it is not relevant.  For the time being I have left
the links to the docker images and setup to the original as they do work.  As I rebuild
and customize I will update this README.

This project was spawned by the desire to [control hubitat and Hubitat Elevation from within Home Assistant][ha-issue].  Since Home Assistant already supports MQTT.  The original authors chose to go and build a bridge between hubitat and MQTT and I have borrowed their work.


# Architecture

![Architecture](https://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgU21hcnRUaGluZ3MgPC0-IE1RVFQgCgpwYXJ0aWNpcGFudCBaV2F2ZSBMaWdodAoKAAcGTW90aW9uIERldGVjdG9yLT5TVCBIdWI6ABEIRXZlbnQgKFotV2F2ZSkKABgGACEFTVFUVEJyaWRnZSBBcHA6IERldmljZSBDaGFuZ2UAMAhHcm9vdnkAMwUAIg4AMxAAOAY6IE1lc3NhADYKSlNPTgAuEABjBi0-AHYLU2VyADkGAHAVUkVTVCkKAB0SAD0GIEJyb2tlcgCBaQk9IHRydWUgKE1RVFQpCgAyBQAcBwBdFgCCSgUgPSAib24iAC4IAFgUAIFaFgCBFhsAgWAWAIJnEwCCESMAgmoIAINWBVR1cm4AgTAHT24AgxcNAINXBQCEGwsAgVYIT24Ag3oJ&s=default)

# MQTT Events

Events about a device (power, level, switch) are sent to MQTT using the following format:

```
{PREFACE}/{DEVICE_NAME}/${ATTRIBUTE}
```
__PREFACE is defined as "hubitat" by default in your configuration__

For example, my Dimmer Z-Wave Switch is called "Family Room Lights" in Hubitat Elevation.  The following topics are published:

```
# Brightness (0-99)
hubitat/Family Room Lights/level
# Switch State (on|off)
hubitat/Family Room Lights/switch
```

The Bridge also subscribes to changes in these topics, so that you can update the device via MQTT.

```
$ mqtt pub -t 'hubitat/Family Room Lights/switch'  -m 'off'
# Light goes off in Hubitat Elevation
```

# Configuration

The bridge has one yaml file for configuration:

```
---
mqtt:
    # Specify your MQTT Broker URL here
    host: mqtt://localhost
    # Example from CloudMQTT
    # host: mqtt:///m10.cloudmqtt.com:19427

    # Preface for the topics $PREFACE/$DEVICE_NAME/$PROPERTY
    preface: hubitat

    # The write and read suffixes need to be different to be able 
    # to differentiate when state comes from Hubitat Elevation or 
    # when its coming from the physical device/application

    # Suffix for the topics that receive state from Hubitat Elevation $PREFACE/$DEVICE_NAME/$PROPERTY/$STATE_READ_SUFFIX
    # Your physical device or application should subscribe to this topic to get updated status from Hubitat Elevation
    # state_read_suffix: state

    # Suffix for the topics to send state back to Hubitat Elevation $PREFACE/$DEVICE_NAME/$PROPERTY/$STATE_WRITE_SUFFIX
    # your physical device or application should write to this topic to update the state of Hubitat Elevation devices that support setStatus
    # state_write_suffix: set_state

    # Suffix for the command topics $PREFACE/$DEVICE_NAME/$PROPERTY/$COMMAND_SUFFIX
    # command_suffix: cmd

    # Other optional settings from https://www.npmjs.com/package/mqtt#mqttclientstreambuilder-options
    # username: AzureDiamond
    # password: hunter2

    # MQTT retains state changes be default, retain mode can be disabled:
    # retain: false

# Port number to listen on
port: 8080

```

# Installation

There are two ways to use this, Docker (self-contained) or NPM (can run on Raspberry Pi).

## Docker

Docker will automatically download the image, but you can "install" it or "update" it via `docker pull`:
```
$ docker pull stjohnjohnson/smartthings-mqtt-bridge
```

To run it (using `/opt/mqtt-bridge` as your config directory and `8080` as the port):
```
$ docker run \
    -d \
    --name="mqtt-bridge" \
    -v /opt/mqtt-bridge:/config \
    -p 8080:8080 \
    stjohnjohnson/smartthings-mqtt-bridge
```

To restart it:
```
$ docker restart mqtt-bridge
```

## NPM

To install the module, just use `npm`:
```
$ npm install -g smartthings-mqtt-bridge
```

If you want to run it, you can simply call the binary:
```
$ smartthings-mqtt-bridge
Starting SmartThings MQTT Bridge - v1.1.3
Loading configuration
No previous configuration found, creating one
```

Although we recommend using a process manager like [PM2][pm2]:
```
$ pm2 start smartthings-mqtt-bridge
[PM2] Starting smartthings-mqtt-bridge in fork_mode (1 instance)
[PM2] Done.
┌─────────────────────────┬────┬──────┬───────┬────────┬─────────┬────────┬────────────┬──────────┐
│ App name                │ id │ mode │ pid   │ status │ restart │ uptime │ memory     │ watching │
├─────────────────────────┼────┼──────┼───────┼────────┼─────────┼────────┼────────────┼──────────┤
│ smartthings-mqtt-bridge │ 1  │ fork │ 20715 │ online │ 0       │ 0s     │ 7.523 MB   │ disabled │
└─────────────────────────┴────┴──────┴───────┴────────┴─────────┴────────┴────────────┴──────────┘

$ pm2 logs smartthings-mqtt-bridge
smartthings-mqtt-bridge-1 (out): info: Starting SmartThings MQTT Bridge - v1.1.3
smartthings-mqtt-bridge-1 (out): info: Loading configuration
smartthings-mqtt-bridge-1 (out): info: No previous configuration found, creating one

$ pm2 restart smartthings-mqtt-bridge
```

## Usage
1. Customize the MQTT host
    ```
    $ vi config.yml
    # Restart the service to get the latest changes
    ```

2. Install the [Driver][dt] in the Hubitat Web Interface [Drivers Code] using "New Driver"

3. Add the "MQTT Device" device in the [Devices]. Enter MQTT Device (or whatever) for the name. Select "MQTT Bridge" for the type. The other values are up to you.
4. Configure the "MQTT Device" in the [Devices] with the IP Address, Port, and MAC Address of the machine running the Docker container
4. Install the [App][app] in [Apps Code] using "New App"
5. Enable the App in [Apps] using "Load New App" and select MQTT Bridge
6. Click MQTT Bridge in the list of Apps
7. At the top of notification (this is to be updated)
8. Configure the devices you want to send to the MQTT Bridge.
9. At the bottom select the MQTT Bridge to send events to.
10. Save

## Advanced
### Docker Compose

If you want to bundle everything together, you can use [Docker Compose][docker-compose].

Just create a file called `docker-compose.yml` with this contents:
```yaml
mqtt:
    image: matteocollina/mosca
    ports:
        - 1883:1883

mqttbridge:
    image: stjohnjohnson/smartthings-mqtt-bridge
    volumes:
        - ./mqtt-bridge:/config
    ports:
        - 8080:8080
    links:
        - mqtt

homeassistant:
    image: balloob/home-assistant
    ports:
        - 80:80
    volumes:
        - ./home-assistant:/config
        - /etc/localtime:/etc/localtime:ro
    links:
        - mqtt
```

This creates a directory called `./mqtt-bridge/` to store configuration for the bridge.  It also creates a directory `./home-assistant` to store configuration for HA.



 [dt]: https://github.com/jeubanks/hubitat-mqtt-bridge/blob/master/drivers/hubitat-mqtt-bridge-driver.groovy
 [app]: https://github.com/jeubanks/hubitat-mqtt-bridge/blob/master/apps/hubitat-mqtt-bridge-app.groovy
 [ha-issue]: https://github.com/balloob/home-assistant/issues/604
 [docker-compose]: https://docs.docker.com/compose/
