# tvheadend-mqtt

## Overview
An extension to [Tvheadend](https://tvheadend.org/) that exposes the Tvheadend API via MQTT.

Key features:
- Access to the Tvheadend API via MQTT
- Simple setup with lightweight Docker container
- Runs on any host on the network
- Customizable and extensible
- Optional features:
  - Publishing triggered by Tvheadend
  - [FHEM](https://fhem.de) integration

## How it works
A client on the network publishes a command request to the MQTT Topic `${MQTT_TOPIC_PREFIX}/request`.
Example for command `subscriptions` and MQTT broker `broker`:
    $ mosquitto_pub -h broker -t tvheadend/request -m subscriptions

`tvheadend-mqtt` publishes the response to the MQTT topic `${MQTT_TOPIC_PREFIX}/`*command*. Example:
    $ mosquitto_sub -h broker -t tvheadend/# -v
    tvheadend/subscriptions {"entries":[],"totalCount":0}

### Built-in commands:
- `connections`
- `subscriptions`
- `upcoming`
- `finished`
- `storage` (requires Tvheadend configuration)

### Plugins
Additional commands may be added by creating a file `plugins` and mounting it under `/app/plugins` into the container.
See the file `plugins.example` for details.

## Setup
Prerequisites:
* A running Tvheadend service
* A running MQTT broker

### Server
Dependencies:
* Docker
* optional and recommended: docker-compose
On Debian based systems, dependencies may be installed using the command
    $ sudo apt-get install docker-ce docker-compose

Installation:
    $ mkdir tvheadend-mqtt
    $ cd tvheadend-mqtt
    $ wget https://github.com/git-developer/tvheadend-mqtt/raw/master/docker-compose.yml
    # edit docker-compose.yml - see configuration section
    $ sudo docker-compose up -d

Test 1: Print status to shell
    $ sudo docker exec tvheadend-mqtt /app/bin/main get_subscriptions
    {"entries":[],"totalCount":0}

Test 2: Publish status via MQTT
    $ sudo docker exec tvheadend-mqtt /app/bin/main publish subscriptions
    22:58:53 publish subscriptions

### Client(s)
Dependencies:
* An MQTT client
On Debian based systems, dependencies may be installed using the command
    $ sudo apt-get install mosquitto-clients

Test: Subscribe and request a command
    $ mosquitto_sub -h broker -t tvheadend/# -v &
    $ mosquitto_pub -h broker -t tvheadend/request -m subscriptions
    tvheadend/request subscriptions
    tvheadend/subscriptions {"entries":[],"totalCount":0}
    $ kill "$!"

## Configuration 
### Environment variables

| Variable                 | Description            | Required? |  Default     |
| ------------------------ | ---------------------- | --------- | ------------ |
| `TVHEADEND_USER`         | Tvheadend username     | required  |              |
| `TVHEADEND_PASSWORD`     | Tvheadend password     | required  |              |
| `TVHEADEND_HOST`         | Tvheadend hostname     | required  |              |
| `MQTT_BROKER_HOSTNAME`   | MQTT broker hostname   | required  |              |
| `TVHEADEND_HTTP_PORT`    | Tvheadend HTTP port    | optional  | `9981`       |
| `TVHEADEND_HTTP_URL`     | Tvheadend API URL      | optional  | `http://${TVHEADEND_HOST}:${TVHEADEND_HTTP_PORT}` |
| `TVHEADEND_HTTP_TIMEOUT` | HTTP Timeout (seconds) | optional  | 15           |
| `MQTT_TOPIC_PREFIX`      | Prefix for MQTT topic  | optional  | `tvheadend`  |
| `TZ`                     | Timezone               | optional  | none         |

### Volumes
The directory `/app/markers` inside the docker container may be mounted to a directory on the host.
A change to one of the following files within this directory triggers an MQTT publish:
* `recording-post-process`
* `recording-post-remove`
* `recording-pre-process`

The amount of published information is built-in and may be customized with the `plugins` file.
See section *Publishing triggered by Tvheadend* for details.

### Example for a minimal configuration
    ---
    version: "2"
    services:
       tvheadend-mqtt:
        image: dockomento/tvheadend-mqtt:latest
        container_name: tvheadend-mqtt
        environment:
          - TZ=Europe/Berlin
          - TVHEADEND_USER=hts
          - TVHEADEND_PASSWORD=hts
          - TVHEADEND_HOST=localhost
          - MQTT_BROKER_HOSTNAME=localhost
        # volumes:
        #   - /home/hts/markers:/app/markers

## Optional features
### Publishing triggered by Tvheadend
Requirement for this feature is a shared *marker directory* that is writeable by Tvheadend and readable by `tvheadend-mqtt`.
In this section, the marker directory is assumed to be `/home/hts/markers`.

Tvheadend Configuration: Recording
- Pre-processor command:  `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-pre-process"`
- Post-processor command: `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-post-process"`
- Post-remove command:    `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-post-remove"`

`docker-compose.yml`: mount marker directory to `/app/markers`
        volumes:
          - /home/hts/markers:/app/markers

The default handler publishes built-in commands via MQTT. This behavior may be customized by
defining the function `handle_notify()` in the plugins file. See the file `plugins.example` for an example.


### FHEM integration
```
define mqtt_tvheadend MQTT_DEVICE
attr   mqtt_tvheadend alias Tvheadend
attr   mqtt_tvheadend subscribeReading_connections   tvheadend/connections
attr   mqtt_tvheadend subscribeReading_subscriptions tvheadend/subscriptions
attr   mqtt_tvheadend subscribeReading_upcoming      tvheadend/upcoming
attr   mqtt_tvheadend subscribeReading_finished      tvheadend/finished
attr   mqtt_tvheadend subscribeReading_storage       tvheadend/storage
attr   mqtt_tvheadend event-on-change-reading .*

define expandjson_mqtt_tvheadend expandJSON mqtt_tvheadend..+:.\{.*\}
attr   expandjson_mqtt_tvheadend alias Tvheadend: JSON
attr   expandjson_mqtt_tvheadend addReadingsPrefix 1

define group_mqtt_tvheadend readingsGroup \
 mqtt_tvheadend:<Recorder>,upcoming_entries_01_sched_status,<>,upcoming:t\
 mqtt_tvheadend:<Subscriptions>,subscriptions_totalCount,<>,subscriptions:t\
 mqtt_tvheadend:@2,<Storage>,(storage_/recordings)_percent,#1_available,storage:t\
 mqtt_tvheadend:@2,<>,(upcoming_entries_..)_disp_title,#1_disp_extratext,#1_start\
 mqtt_tvheadend:@2,<>,(finished_entries_..)_disp_title,#1_disp_extratext,#1_start
attr   group_mqtt_tvheadend nonames 1
attr   group_mqtt_tvheadend nameStyle { Upcoming => 'style="font-weight:bold;; text-align:center"', Finished => 'style="font-weight:bold;; text-align:center"' }
attr   group_mqtt_tvheadend valueFormat { if ($READING =~ ".+_start\$") { $VALUE = substr(FmtDateTime($VALUE), 0, 16) } elsif ($READING =~ ".+_percent\$") { "$VALUE% used" } elsif ($READING =~ ".+_available\$") { "$VALUE available" } }
attr   group_mqtt_tvheadend valueColumns { if ($READING =~ '(Finished|Upcoming)') {'colspan="5"'} }
attr   group_mqtt_tvheadend valueIcon {'upcoming_entries_01_sched_status.scheduled' => 'ios-on-for-timer-green', 'upcoming_entries_01_sched_status.recording' => '10px-kreis-rot'}
attr   group_mqtt_tvheadend valueStyle { if ($VALUE =~ '\d+%') { "style=\"width:100px;; text-align:center;; border: 1px solid #ccc;; background:-webkit-linear-gradient(left, green $VALUE, rgba(0,0,0,0) $VALUE)\"" }}
attr   group_mqtt_tvheadend alias Tvheadend
```
