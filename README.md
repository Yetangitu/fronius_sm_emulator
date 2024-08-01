# fronius_sm_emulator
MQTT to Modbus TCP bridge, emulating a Fronius "Smart Meter"

Emulates a Fronius Smart Meter using a MQTT data source (e.g. a utility meter with a P1 port reader), producing output through Modbus TCP. This 'virtual meter' can be added as a 'Fronius Smart Meter (TCP)' power meter to Fronius inverters, eliminating the need to install a separate smart meter.

![image](https://github.com/user-attachments/assets/0021fcf6-5d76-45a2-af34-cb9f75e87cce)
![image](https://github.com/user-attachments/assets/519f3a76-b3a0-425c-8732-e8ea7706089b)

## Installation
This thing is written in Python and as such liable to be subject to bitrot. As it stands it works in Debian Stable using the following dependencies:

- python3-pymodbus (3.0.0)
- python3-paho-mqtt (1.6.1)

If you happen to have `paho-mqtt` v2 installed you'll need to edit `fronius_sm_emulator` to comment out the call to mqtt.Client and remove the `#` from the line above it since the call signature changed.

Copy `fronius_sm_emulator` to a directory of choice - `/usr/local/bin` comes to mind - and make sure the file is executable. If you use _systemd_ you can use the included unit file (`fronius_sm_emulator.service`) by copying it to `/etc/systemd/system`. Edit it to configure the `Environment=` lines to your installation or copy and edit the environment file (`fronius_sm_emulator.env`) to `/etc/defaults/fronius_sm_emulator` and enable the `EnvironmentFile=/etc/default/fronius_sm_emulator` line.

## Configuration

This tool is configured through environment variables:

```bash
    MQTT_USERNAME='' # MQTT broker username
    MQTT_PASSWORD='' # MQTT broker password
    MQTT_ADDRESS=''  # FQDN or IP address for MQTT broker
    MQTT_PORT='1883' # 1833 is the default MQTT port
    # adjust the following MQTT topics to the intended data source
    MQTT_TOPIC_MOMENTARY_IMPORT='p1-reader-1/sensor/momentary_active_import/state'
    MQTT_TOPIC_MOMENTARY_EXPORT='p1-reader-1/sensor/momentary_active_export/state'
    MQTT_TOPIC_TOTAL_IMPORT='p1-reader-1/sensor/cumulative_active_import/state'
    MQTT_TOPIC_TOTAL_EXPORT='p1-reader-1/sensor/cumulative_active_export/state'
    CORRECTION_FACTOR='1000'    # correction factor to produce W/Wh for Modbus output
    MODBUS_PORT='502'           # defaults to 502
    MODBUS_DEVICE_ADDRESS='240' # defaults to 240
    LOG_LEVEL='ERROR'           # defaults to 'ERROR'
```

Add these parameters to an environment file (e.g. /etc/default/fronius_sm_emulator) which is
sourced before starting this script or use Environment= entries in a systemd service file:

```
    [Unit]
    Description = Fronius Smart Meter emulator (MQTT to Modbus TCP bridge)
    
    [Service]
    #EnvironmentFile=/etc/default/fronius_sm_emulator
    # ...or use Environment= lines:
    #Environment=MQTT_USERNAME=''
    #Environment=MQTT_PASSWORD=''
    #Environment=MQTT_ADDRESS=''
    #Environment=MQTT_PORT='1883'
    #Environment=MQTT_TOPIC_MOMENTARY_IMPORT='p1-reader-1/sensor/momentary_active_import/state'
    #Environment=MQTT_TOPIC_MOMENTARY_EXPORT='p1-reader-1/sensor/momentary_active_export/state'
    #Environment=MQTT_TOPIC_TOTAL_IMPORT='p1-reader-1/sensor/cumulative_active_import/state'
    #Environment=MQTT_TOPIC_TOTAL_EXPORT='p1-reader-1/sensor/cumulative_active_export/state'
    #Environment=CORRECTION_FACTOR='1000'
    #Environment=MODBUS_PORT='502'
    #Environment=MODBUS_DEVICE_ADDRESS='240'
    #Environment=LOG_LEVEL='ERROR'
    ExecStart=/usr/local/bin/fronius_sm_emulator
    # root is needed when using the default Modbus port (502)
    User=root
    # The inverter sometimes stops reading meter output after about a day; it shows the meter
    # as being present and responding but stops reading from it. Restarting the
    # bridge restores functionality. Enable the following two lines to have systemd 
    # restart the service after a given interval, default 1 day (1d).
    #Restart=always
    #RuntimeMaxSec=1d
```

### Correction and conversion of readings
Add the connection details to your MQTT broker - address, username and password. Edit the `MQTT_TOPIC_...` parameters to fit your data source. If you use the included _systemd_ unit file and you copied _fronius_sm_emulator_ to another directory than the one mentioned under `ExecStart=` you'll need to edit that parameter to point to the correct location.
The `CORRECTION_FACTOR` parameter is used to convert meter readings (often in _kW_/_kWh_/_kVA_/_kVAr/...) to _W_/_Wh (etc.), adjust it when needed.
As it stands the bridge is made to work in combination with a P1 reader connected to a utility meter (model _Sanxing SX6x1 (SxxU1x)_ which produces `Ascii OBIS` output on the P1 port) which produces two separate readings for active power import and active power export. The SunSpec model used by the Fronius model which this bridge emulates uses a single reading which goes negative when power is exported to the grid. This difference is handled in the `update_datablock()` function:
```
...
electrical_power_corr = float(momentary_import - momentary_export)*CORRECTION_FACTOR
...
```
If your data source uses a single reading for power import/export you'll need to adapt this function.

## Inspiration

Modbus data model according to Fronius Meter_Register_Map_Float_v1.0

Based on https://www.photovoltaikforum.com/thread/185108-fronius-smart-meter-tcp-protokoll

