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
Clients on the network may publish a command request to the MQTT Topic `${MQTT_TOPIC_PREFIX}/request`.
Example for command `subscriptions` and MQTT broker `broker`:

    $ mosquitto_pub -h broker -t tvheadend/request -m subscriptions

`tvheadend-mqtt` then publishes the response to the MQTT topic `${MQTT_TOPIC_PREFIX}/`*command*. Example:

    $ mosquitto_sub -h broker -t tvheadend/# -v
    tvheadend/subscriptions {"entries":[],"totalCount":0}

### Built-in commands
- `connections`
- `subscriptions`
- `upcoming`
- `finished`
- `storage` (requires Tvheadend configuration, see *Publishing triggered by Tvheadend*)

### Plugins
Additional commands may be added by creating a file `plugins`
and mounting it under `/app/plugins` into the container.
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

Edit `docker-compose.yml` - see configuration section

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

| Variable                 | Description            | Required? |  Default    |
| ------------------------ | ---------------------- | --------- | ----------- |
| `TVHEADEND_USER`         | Tvheadend username     | required  |             |
| `TVHEADEND_PASSWORD`     | Tvheadend password     | required  |             |
| `TVHEADEND_HOST`         | Tvheadend hostname     | required  |             |
| `MQTT_BROKER_HOSTNAME`   | MQTT broker hostname   | required  |             |
| `TVHEADEND_HTTP_PORT`    | Tvheadend HTTP port    | optional  | `9981`      |
| `TVHEADEND_HTTP_URL`     | Tvheadend API URL      | optional  | `http://${TVHEADEND_HOST}:${TVHEADEND_HTTP_PORT}` |
| `TVHEADEND_HTTP_TIMEOUT` | HTTP Timeout (seconds) | optional  | 15          |
| `MQTT_TOPIC_PREFIX`      | Prefix for MQTT topic  | optional  | `tvheadend` |
| `MQTT_PUBLISH_OPTIONS`   | MQTT publish options   | optional  |             |
| `MQTT_SUBSCRIBE_OPTIONS` | MQTT subscribe options | optional  |             |
| `TZ`                     | Timezone               | optional  |             |

### Example: minimal configuration
    ---
    version: "2"
    services:
       tvheadend-mqtt:
        image: ckware/tvheadend-mqtt:latest
        environment:
          - TVHEADEND_USER=hts
          - TVHEADEND_PASSWORD=hts
          - TVHEADEND_HOST=localhost
          - MQTT_BROKER_HOSTNAME=localhost

### Example: standard configuration
    ---
    version: "2"
    services:
       tvheadend-mqtt:
        image: ckware/tvheadend-mqtt:latest
        container_name: tvheadend-mqtt
        environment:
          - TZ=Europe/Berlin
          - TVHEADEND_USER=hts
          - TVHEADEND_PASSWORD=hts
          - TVHEADEND_HOST=localhost
          - MQTT_BROKER_HOSTNAME=localhost
        volumes:
          # support for publishing triggered by Tvheadend (optional)
          - /home/hts/markers:/app/markers
          # support for user-defined plugins (optional)
          - ./plugins:/app/plugins
        restart: unless-stopped

## Optional features
### Publishing triggered by Tvheadend
Requirement for this feature is a shared *marker directory*
that is writeable by Tvheadend and readable by `tvheadend-mqtt`.
In this section, the following assumptions are made:

- marker directory: `/home/hts/markers`
- directory for Tvheadend recordings: `/recordings`

1. Configure Tvheadend to write marker files:
    Login to Tvheadend, go to *Configuration / Recording / Miscellaneous Settings* and set at least one of
    * Pre-processor command:  `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-pre-process"`
    * Post-processor command: `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-post-process"`
    * Post-remove command:    `/bin/sh -c "/bin/df -P -h /recordings >/home/hts/markers/recording-post-remove"`
1. Extend `docker-compose.yml`:
    Mount the marker directory to `/app/markers`
    ```
        volumes:
           - /home/hts/markers:/app/markers
    ```
1. (Re-)Create the Docker container
    ```
    sudo docker-compose down; sudo docker-compose up -d'
    ```

The built-in handler watches the following files within the marker directory
for changes and publishes pre-defined commands via MQTT:

* `recording-post-process`
* `recording-post-remove`
* `recording-pre-process`

The built-in handler (`/app/bin/handle_notify`) may be replaced
by a custom handler using the `plugins` file. See the file `plugins.example` for an example.


### FHEM integration
This section contains examples for a FHEM configuration using built-in modules only.

#### Using a MQTT2_DEVICE
- `MQTT2_DEVICE` registers FHEM as MQTT subscriber for Tvheadend topics and creates FHEM readings from published JSON messages
- `readingsGroup` visualizes the Tvheadend info in a table
```
define mqtt_tvheadend MQTT2_DEVICE
attr   mqtt_tvheadend readingList tvheadend/.* { json2nameValue($EVENT, "$1_") if ("$TOPIC" =~ '^.*/([^/]+)$') }
attr   mqtt_tvheadend event-on-change-reading .*

define group_mqtt_tvheadend readingsGroup \
 mqtt_tvheadend:<Recorder>,upcoming_entries_1_sched_status,<>,upcoming_entries_1_sched_status:t\
 mqtt_tvheadend:<Subscriptions>,subscriptions_totalCount,<>,subscriptions_totalCount:t\
 mqtt_tvheadend:@2,<Storage>,(storage_/recordings)_percent,#1_available,storage_/recordings_size:t\
 <Upcoming>\
 mqtt_tvheadend:@1,(upcoming_entries_..?)_disp_title,#1_disp_extratext,#1_start\
 <Finished>\
 mqtt_tvheadend:@1,(finished_entries_..?)_disp_title,#1_disp_extratext,#1_start
attr   group_mqtt_tvheadend nonames 1
attr   group_mqtt_tvheadend nameStyle    { Upcoming => 'style="font-weight:bold;; text-align:center"', Finished => 'style="font-weight:bold;; text-align:center"' }
attr   group_mqtt_tvheadend valueFormat  { if ($READING =~ ".+_start\$") { $VALUE = substr(FmtDateTime($VALUE), 0, 16) } elsif ($READING =~ ".+_percent\$") { "$VALUE% used" } elsif ($READING =~ ".+_available\$") { "$VALUE free" } }
attr   group_mqtt_tvheadend valueColumns { if ($READING =~ '(Upcoming|Finished)') {'colspan="5"'} elsif ($READING =~ '.+_disp_extratext') {'colspan="2"'} }
attr   group_mqtt_tvheadend valueIcon    {'upcoming_entries_1_sched_status.scheduled' => 'ios-on-for-timer-green', 'upcoming_entries_1_sched_status.recording' => '10px-kreis-rot'}
attr   group_mqtt_tvheadend valueStyle   { if ($VALUE =~ '(\d+)%') { my $color = ('green', 'yellow', 'red')[List::Util::min(2,List::Util::max(0,int($1-60)/20))]; "style=\"width:100px; text-align:center; color:black; background:linear-gradient(to right, $color $VALUE, #ccc)\"" } }
attr   group_mqtt_tvheadend alias Tvheadend
```

#### Using a MQTT_DEVICE
- `MQTT_DEVICE` registers FHEM as MQTT subscriber for Tvheadend topics
- `expandJSON` creates FHEM readings from published JSON messages
- `readingsGroup` visualizes the Tvheadend info in a table
```
define mqtt_tvheadend MQTT_DEVICE
attr   mqtt_tvheadend alias Tvheadend: MQTT
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
attr   group_mqtt_tvheadend nameStyle    { Upcoming => 'style="font-weight:bold;; text-align:center"', Finished => 'style="font-weight:bold;; text-align:center"' }
attr   group_mqtt_tvheadend valueFormat  { if ($READING =~ ".+_start\$") { $VALUE = substr(FmtDateTime($VALUE), 0, 16) } elsif ($READING =~ ".+_percent\$") { "$VALUE% used" } elsif ($READING =~ ".+_available\$") { "$VALUE available" } }
attr   group_mqtt_tvheadend valueColumns { if ($READING =~ '(Finished|Upcoming)') {'colspan="5"'} }
attr   group_mqtt_tvheadend valueIcon    {'upcoming_entries_01_sched_status.scheduled' => 'ios-on-for-timer-green', 'upcoming_entries_01_sched_status.recording' => '10px-kreis-rot'}
attr   group_mqtt_tvheadend valueStyle   { if ($VALUE =~ '\d+%') { "style=\"width:100px;; text-align:center;; border: 1px solid #ccc;; background:-webkit-linear-gradient(left, green $VALUE, rgba(0,0,0,0) $VALUE)\"" }}
attr   group_mqtt_tvheadend alias Tvheadend
```
