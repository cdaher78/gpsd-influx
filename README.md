# Introduction
This script can be run as a daemon to collect information from a GPS and push it into an Influx Database.  This can be useful for tracking or for monitoring GPS drift.

![grafana dashboard](https://github.com/cdaher78/gpsd-influx/blob/dev-influxv2/grafana.png)

* Dashboard Source: https://github.com/cdaher78/gpsd-influx/blob/dev-influxv2/gpsd-dashboard.json
# Reference
* JSON output of gpspipe: https://gpsd.gitlab.io/gpsd/gpsd_json.html

* GPS on Raspberry Pi: https://maker.pro/raspberry-pi/tutorial/how-to-use-a-gps-receiver-with-raspberry-pi-4

# Requirements
* A serial or USB GPS
* A dedicated computer to run the daemon on (I'm using a Raspberry Pi 4B and a serial GPS)
* gpsd (https://gpsd.gitlab.io/gpsd/)
* InfluxDB (https://www.influxdata.com/)

# Optional
* Grafana for visualizing the data (https://grafana.com/)

# Installation
## InfluxDB
Installation instructions for InfluxDB: https://docs.influxdata.com/influxdb/v2.3/install/

Once you have Influx installed, create your organization, a bucket and an API token:

## gpsd

Debian based install
```
apt update
apt install -y gpsd
```

Once it is intalled, make sure to edit */etc/default/gpsd* and change the *DEVICES* line to match the device for your GPS, ex:
```
# Default settings for the gpsd init script and the hotplug wrapper.

# Start the gpsd daemon automatically at boot time
START_DAEMON="true"

# Use USB hotplugging to add new USB devices automatically to the daemon
USBAUTO="true"

# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/ttyUSB0"

# If using a serial GPS
DEVICES="dev/ttyAMA0"
USBAUTO"false"

# Other options you want to pass to gpsd
GPSD_OPTIONS=""
```

Enable and start gpsd
```
systemctl enable gpsd.sock
systemctl start gpsd.sock
```

If your gps is connected you can test that it is working by running the following command:
```
gpspipe -w -n 5
{"class":"VERSION","release":"3.16","rev":"3.16-4","proto_major":3,"proto_minor":11}
{"class":"DEVICES","devices":[{"class":"DEVICE","path":"/dev/ttyUSB0","driver":"SiRF","subtype":"9\u0006GSD4e_4.1.2-B2_RPATCH.02-F-GPS-4R-1301151 01/17/2013 017","activated":"2019-05-19T14:34:37.601Z","flags":1,"native":1,"bps":4800,"parity":"N","stopbits":1,"cycle":1.00}]}
{"class":"WATCH","enable":true,"json":true,"nmea":false,"raw":0,"scaled":false,"timing":false,"split24":false,"pps":false}
{"class":"TPV","device":"/dev/ttyUSB0","mode":3,"time":"2019-05-19T14:34:39.000Z","ept":0.005,"lat":45.xxxxxxxxx,"lon":-73.xxxxxxxxx,"alt":42.110,"epx":8.341,"epy":14.615,"epv":32.200,"track":0.0000,"speed":0.000,"climb":0.000,"eps":29.23,"epc":64.40}
```

On the TPV line you should see your latitude and longitude displayed.

# gpsd-influx script
Now that gpsd is installed and working, you can install the script.

```
git clone -b dev-influxv2 https://github.com/cdaher78/gpsd-influx.git /opt/gpsd-influx
chmod a+x /opt/gpsd-influx/gpsd-influx.sh
```

Edit the script and make sure to change the variables at the top to match your configuration:
```
# Your Influxdb Server
INFLUX_URL="Your Influxdb Server"
# Your Influxdb Organization
YOUR_ORG="Your Influxdb Organization"
# Your Influxdb Bucket
YOUR_BUCKET="Your Influxdb Bucket"
# Your API Token
YOUR_API_TOKEN="Your API Token"
# Number of seconds between updates
update_interval=10
```

Once that is complete, you can test the script with the debug flag:
```
/opt/gpsd-influx/gpsd-influx.sh -d
--------------------------------------------------------------------------------
TPV
{
    "alt": 38.627,
    "class": "TPV",
    "climb": 0.0,
    "device": "/dev/ttyUSB0",
    "epc": 64.4,
    "eps": 29.23,
    "ept": 0.005,
    "epv": 32.2,
    "epx": 8.341,
    "epy": 14.615,
    "lat": 45.xxxxxxxxx,
    "lon": -73.xxxxxxxxx,
    "mode": 3,
    "speed": 0.0,
    "time": "2019-05-19T14:40:12.000Z",
    "track": 0.0
}
--------------------------------------------------------------------------------
Sending to Influx:
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=alt value=38.627
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=climb value=0.0
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=epc value=64.4
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=eps value=29.23
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=ept value=0.005
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=epv value=32.2
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=epx value=8.341
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=epy value=14.615
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=lat value=45.xxxxxxxxx
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=lon value=-73.xxxxxxxxx
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=mode value=3
gpsd,host=pi-gpsd,device="/dev/ttyUSB0",tpv=speed value=0.0
--------------------------------------------------------------------------------

```
If you didn't get any errors you should be good to go to setup the script to run as a daemon.

```
cat <<EOF >> /etc/systemd/system/gpsd-influx.service

[Unit]
Description=GPSD to Influx
After=syslog.target

[Service]
ExecStart=/opt/gpsd-influx/gpsd-influx.sh
KillMode=process
Restart=on-failure
User=root

StandardOutput=append:/var/log/gpsd-influx.log
StandardError=append:/var/log/gpsd-influx.log

[Install]
WantedBy=multi-user.target
EOF
```

Now you can enable and start the daemon:
```
systemctl enable gpsd-influx.service
systemctl start gpsd-influx.service
```
