Fortigate Syslog Configuration
======================================

### UDP

Configuring syslog is widely used across companies. Perhaps, even if
they may have other syslog servers already configured, this **does not**
mean the firewall is configured properly.

It is best to isolate this traffic from any other devices.

UDP Syslog messages are limited to 64 KB. If the message is longer, data
may be truncated.

The following configuration is to be added using the console/ssh in the
Fortigate appliance.

You must change the following values in order for this to work:

- LOGSTASH_IP → IP address of the Logstash server (most likely the agent's IP)
- FGT_IP → Private IP address of the appliance.

### CLI

    config log syslogd setting
        set status enable
        set server "LOGSTASH_IP"
        set mode udp
        set port 5514
        set facility local7
        set source-ip "FGT_IP"
        set format default
        set priority default
        set max-log-rate 10
    end


# Logstash Configuration

### 10-input_syslog

Receives syslog logs and populates `event.module`:
```
add_field => {"[event][module]" => "fortigate"}
```

### 20-observer-enrichment
Fields are mapped based on two dictionaries from the following path:
`<project_root>/logstash/pipeline/dictionaries`

- `host.yml` enriches `firewall.*`
- `geo.yml` enriches `observer.*` which contains geo-location details and more.

More details about dictionaries at the end of this file.

### 30-kv_syslog

Splits the original log into key-value pairs, and sets the timestamp. Timezone is also dynamically obtained the `geo.yml` dictionary.

### 40-fortigate_2_ecs

- Translates Fortinet fields to ECS. 
- Renames Fortinet fields that overlaps with ECS
- Validates nulls IP fields to prevent ingestion errors
- Populates other ECS fields based on ECS recommendations. (related ip, source.address, event.start, etc.)

### 50-geo_enrichment

The following fields are inspected to decide whether they are public or private address with the help of [.locality](https://github.com/elastic/ecs/pull/288) fields:

- `source.ip` 
- `source.nat.ip` 
- `destination.ip` 
- `destination.nat.ip` 

If they are public, then geo enrichment will be applied.

### 60-logstash_enrichment
Uses the `agent.yml` dictionary to provide details about the logstash instance the logs are coming from.

### 70-drop
Uses the `drop_networks.yml` dictionary to drop any defined CIDRs
from being ingested. This could be for example, your guest networks.

### 80-output
This is the final pipeline which will output with in the following index according to the variables defined under `host.yml`:
- "ecs-`%{[event][module]}`-`%{[organization][name]}`-write"

If one of the variables is not found, logs will be sent to an index named `index_error-write`.


Flow Representation
-------------------

![](assets/directory-structure.png)

Properties Definitions
----------------------

These properties should be inline, the representation below is to
provide a better readability.

**Please note**: Most of the fields are not mandatory and can be *skipped* using a commas: "`,`"

### Validations

Due to the following condition under the pipeline `40-fortigate_2_ecs`: 

`if [devid] != [observer][serial_number] or [observer][name] != [devname]`

These fields must not be skipped: [`serial_number`, `name`]

### Host Dictionary Properties

```
"[observer][ip]" : 
"[observer][name]",
"[observer][hostname]",
"[observer][mac]",
"[observer][product]",
"[observer][serial_number]",
"[observer][type]",
"[observer][vendor]",
"[observer][version]",
"[organization][id]",
"[organization][name]"
```

### Geo Dictionary
```
"[observer][ip]": 
"[observer][geo][city_name]",
"[observer][geo][continent_name]",
"[observer][geo][country_iso_code]",
"[observer][geo][country_name]",
"[observer][geo][location][lon]",
"[observer][geo][location][lat]",
"[observer][geo][name]",
"[observer][geo][region_iso_code]",
"[observer][geo][region_name]",
"[event][timezone]",
"[observer][geo][site]",
"[observer][geo][building]",
"[observer][geo][floor]",
"[observer][geo][room]",
"[observer][geo][rack]",
"[observer][geo][rack_unit]"
```

Tags
----

To better assist if something goes wrong, pipelines will tag the events.

  **Tag**                              **Pipeline**
|   Tag                             | Pipeline
|---                                |--                     |
| syslog                            | 10-input_syslog
| guest_network                     | 70-drop
| error_geo_file                    | 20-observer_enrichment
| send_index_error                  | 20-observer_enrichment
| dictionary_error                  | 40-fortigate_2_ecs
| kv_syslog_grok_failure            | 30-kv_syslog
| setting_default_timezone          | 30-kv_syslog
| error_datafeed_enrichment         | 20-observer_enrichment
| _dateparsefailure_kv_syslog       | 30-kv_syslog
| _dateparsefailure_eventtime       | 40-fortigate_2_ecs
| _source_geoip_lookup_failure      | 50-geo_enrichment
| _destination_geoip_lookup_failure | 50-geo_enrichment

Notes
======================================

### Fortigates using HA

You must add the information of the two hosts to the host.yml
configuration file. Otherwise the tag dictionary_error will be applied.

Reference
=========

<http://docs.fortinet.com/document/fortigate/6.2.0/fortios-log-message-reference/682455/log-messages>
<http://docs.fortinet.com/document/fortigate/6.2.0/fortios-log-message-reference/34314/log-id-numbers>
<http://docs.fortinet.com/document/fortigate/6.2.0/fortios-log-message-reference/656858/log-id-definitions>
<http://docs.fortinet.com/document/fortigate/6.2.0/fortios-log-message-reference/357866/log-message-fields>
