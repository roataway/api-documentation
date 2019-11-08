Overview
========

This guide explains how to use the API that exposes realtime location data for public transport in Chișinău. The highlights:

- It follows the publish-subscribe pattern
- MQTT is used for messaging
- Payloads are in JSON
- Vehicles send telemetry every ~3s
- Community support is available via [chat](https://roataway.zulipchat.com/)


Why is this an open API?
------------------------

Although the requirements for the public transport tracking system did not ask for it, someone went the extra mile to make it so. Public transport software is too important to be left in the hands of a single company. An open protocol enables others to roll out their own products that rely on the data - these can have more features, look nicer, or come up with innovative uses of the data that others have not thought of. Thus, consumers are less likely to end up using an outdated or abandoned application that does not keep up with progress.


Prerequisites
-------------

You will be more productive as you go through this guide, if you familiarize yourself with these first:

- Vehicle metadata, shared via [vehicles.csv](https://github.com/roataway/infrastructure-data/blob/master/vehicles.csv)
- Route metadata, shared via [routes.csv](https://github.com/roataway/infrastructure-data/blob/master/routes.csv)
- A basic understanding of the [MQTT protocol](https://www.hivemq.com/blog/mqtt-essentials-part-1-introducing-mqtt/)


Technical overview
==================

Every subscriber will receive the same data, as soon as a vehicle sends the information. You can have multiple subscriptions simultaneously, as well as run multiple instances of your clients.

```
                            +----------+
+----------+    publish     |          |
|  GPS     |   telemetry    |  MQTT    |
|  tracker +---------------->  broker  |
|          |                |          |
+----------+                |          |
                            |          |
                            +-+--+---+-+
                              |  |   |
            +-----------------+  |   |
            |                    |   +-------------+
            |                    |                 |
+-----------v-----+  +-----------v-----+     +-----v-------+
| MQTT subscriber |  |     Another     |     |     Yet     |
| (your software) |  | MQTT subscriber | ..  |   another   |
|                 |  |                 |     |    MQTT     |
|                 |  |                 |     | subscriber  |
+-----------------+  +-----------------+     +-------------+
```


A subscription requires a topic name, here are the ones you can use:

`telemetry/transport/+`
-----------------------

It distributes raw telemetry coming directly from each vehicle, the last element in the topic name will be the tracker ID. Refer to `vehicles.csv` to link the tracker ID to an actual vehicle.

For example, if the topic is `telemetry/transport/000001`, the tracker ID is `000001`, it is attached to trolleybus `1273` (the transport company itself, e.g., RTEC, refers to this as a "board number").


All messages are JSON strings which contain the following keys:

| Field       | Type  | Notes                                                                  |
|-------------|-------|------------------------------------------------------------------------|
| `latitude`  | float |                                                                        |
| `longitude` | float |                                                                        |
| `direction` | float |                                                                        |
| `speed`     | int   | In km/h                                                                |
| `timestamp` | str   | In UTC, e.g. `2019-08-18T16:42:29Z` the format is `%Y-%m-%dT%H:%M:%SZ` |


Example:

```json
{
    "longitude": 28.887143,
    "latitude": 47.026044,
    "timestamp": "2019-08-18T16:42:29Z",
    "speed": 0,
    "direction": 0.0
}
```

Warning:

The payloads may contain other, undocumented keys - don't count on them.


`telemetry/route/+`
-------------------

It distributes slightly enriched telemetry, grouped by route. The last element in the topic name is the `route_upstream_id` that you will find in `routes.csv`.

For example, if the topic is `telemetry/route/1`, the upstream route ID is `1`, it corresponds to route `30, Aeroport - Piața Marii Adunări Naționale`.

This is what the payloads look like:


| Field       | Type  | Notes                                                                  |
|-------------|-------|------------------------------------------------------------------------|
| `latitude`  | float |                                                                        |
| `longitude` | float |                                                                        |
| `direction` | float |                                                                        |
| `speed`     | int   | In km/h                                                                |
| `timestamp` | str   | In UTC, e.g. `2019-08-18T16:42:29Z` the format is `%Y-%m-%dT%H:%M:%SZ` |
| `board`     | str   | The board number of the vehicle                                        |
| `rtu_id`    | str   | The tracker ID                                                         |
| `route`     | int   | The route upstream ID                                                  |


Example:

```json
{
    "latitude": 46.98579,
    "longitude": 28.857805,
    "timestamp": "2019-10-22T07:06:04Z",
    "speed": 12,
    "direction": 313.5,
    "board": "1308",
    "rtu_id": "0000019",
    "route": 3
}
```

Note:

- If a vehicle is moved from one route to another, its telemetry will be automatically distributed via a topic that corresponds to the new route.
- You can subscribe to individual routes, by using their id in the topic name, e.g., `telemetry/route/1`.

Warning:

The payloads may contain other, undocumented keys - don't count on them.

Give it a try
=============

You can use any MQTT client to subscribe to the topics above and see the live data. In these examples we shall use `mosquitto_sub`, distributed with the Mosquitto broker. On Debian-based systems you can install it with `sudo apt install mosquitto-clients`. Here's how to run it:


- `mosquitto_sub -h opendata.dekart.com -p 1945 -t telemetry/transport/+` - receive raw telemetry
- `mosquitto_sub -h opendata.dekart.com -p 1945 -t telemetry/route/+` - receive route-centric telemetry

You can also try `opendata.dekart.com:1946` for MQTT over plaintext websockets.


Credits
=======

We thank [Dekart](https://dekart.com) for designing a system based on non-proprietary protocols and for opening it up to the public.

References
==========

- MQTT libraries for different [programming languages](https://www.eclipse.org/paho/downloads.php).
- Reference implementation that visualizes the [vehicles on a map](https://roataway.md).
