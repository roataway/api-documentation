Overview
========

*DOCUMENT STATUS*: this text is in an RFC phase (request for comments). Feel free to suggest changes, report inconsistencies or missing use cases. You can do so via the chat mechanism, or by opening an issue on Github.


This guide explains how to use the API that exposes realtime location data for public transport. The highlights:

- It follows the publish-subscribe pattern
- MQTT is used for messaging
- Payloads are in JSON
- Vehicles send telemetry every ~20s
- Community support is available via [chat](https://roataway.zulipchat.com/)

Why is this an open API?
------------------------

Although the requirements for the public transport tracking system did not ask for it, someone went the extra mile to make it so. Public transport software is too important to be left in the hands of a single company. An open protocol enables others to roll out their own products that rely on the data - these can have more features, look nicer, or come up with innovative uses of the data that others have not thought of. Thus, consumers are less likely to end up using an outdated or abandoned application that does not keep up with progress.

Ultimately, we want to devise a straightforward on-boarding process, that would allow stakeholders from other transport networks and cities to provide this service to their passengers with minimal investment of time and resources.

Last, but not least, we want to encourage hobbyists and kids to tinker with it, just because real world data is fun to play with!


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
+----------+    publish     +----------+
|  GPS     |   telemetry    |  MQTT    |
|  tracker +---------------->  broker  |
|          |                |          |
+----------+                +-+--+---+-+
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

Topics
------

A subscription requires a topic name, here are the ones you can use:

- `telemetry/transport/<consumer_group>/<city_id>/<network_id>/<tracker_id>`
- `telemetry/route/<consumer_group>/<city_id>/<network_id>/<route_id>`

Here is what the components mean:

- *consumer_group* - reserved for future use (RFU), currently always set to `chiK8n`
- *city_id* - numeric identifier of a city, for now we only have `1 = Chișinău`
- *network_id* - numeric identifier of a transport network within a city, for now we only have `1 = RTEC`

The following section illustrates actual examples.


`telemetry/transport/chiK8n/1/1/+`
----------------------------------

It distributes telemetry coming from each vehicle in `city_id=1` (Chișinău), belonging to `network_id=` (RTEC). In other words, this is telemetry of each trolleybus. The last element in the topic name will be the `tracker_id`. Refer to `vehicles.csv` to link it to an actual vehicle.

For example, if the topic is `telemetry/transport/chiK8n/1/1/000001`, the `tracker_id` is `000001`, it is attached to trolleybus `1273` (the transport company itself, e.g., RTEC, refers to this as a "board number").

All messages are JSON strings which contain the following keys:

| Field       | Type  | Notes                                                                  |
|-------------|-------|------------------------------------------------------------------------|
| `latitude`  | float |                                                                        |
| `longitude` | float |                                                                        |
| `direction` | int   |                                                                        |
| `speed`     | int   | In km/h                                                                |
| `timestamp` | str   | In UTC, e.g. `2019-08-18T16:42:29Z` the format is `%Y-%m-%dT%H:%M:%SZ` |


Example:

```json
{
    "longitude": 28.887143,
    "latitude": 47.026044,
    "timestamp": "2019-08-18T16:42:29Z",
    "speed": 0,
    "direction": 0
}
```

Warning:

The payloads may contain other, undocumented keys - don't count on them.


`telemetry/route/<consumer_group>/<city_id>/<network_id>/+`
-----------------------------------------------------------

It distributes slightly enriched telemetry, grouped by route. The last element in the topic name is the `route_upstream_id` that you will find in `routes.csv`.

For example, if the topic is `telemetry/route/chiK8n/1/1/2`, the upstream route ID is `2`, it corresponds to route `32, str. 31 August 1989 - com. Stăuceni`, while `city_id=1` and `network_id=1` - so it is Chișinău/RTEC.

This is what the payloads look like:


| Field       | Type  | Notes                                                                  |
|-------------|-------|------------------------------------------------------------------------|
| `latitude`  | float |                                                                        |
| `longitude` | float |                                                                        |
| `direction` | int   |                                                                        |
| `speed`     | int   | In km/h                                                                |
| `timestamp` | str   | In UTC, e.g. `2019-08-18T16:42:29Z` the format is `%Y-%m-%dT%H:%M:%SZ` |
| `board`     | str   | The board number of the vehicle                                        |
| `rtu_id`    | str   | The tracker ID                                                         |
| `route`     | str   | The route name (not to be confused with `route_upstream_id`!)          |


Example:

```json
{
    "latitude": 46.98579,
    "longitude": 28.857805,
    "timestamp": "2019-10-22T07:06:04Z",
    "speed": 12,
    "direction": 313,
    "board": "1308",
    "rtu_id": "0000019",
    "route": "3"
}
```

Note:

- If a vehicle is moved from one route to another, its telemetry will be automatically distributed via a topic that corresponds to the new route.
- You can subscribe to individual routes, by using their id in the topic name, e.g., `telemetry/route/chiK8n/1/1/2`, or to all routes in the given city and network, e.g. `telemetry/route/chiK8n/1/1/+`.
- `board`, `rtu_id` and `route` are usually numerical values, but they can also contain letters, treat them as strings!

Warning:

The payloads may contain other, undocumented keys - don't count on them.

Give it a try
=============

You can use any MQTT client to subscribe to the topics above and see the live data. In these examples we shall use `mosquitto_sub`, distributed with the Mosquitto broker. On Debian-based systems you can install it with `sudo apt install mosquitto-clients`. Here's how to run it:


- `mosquitto_sub -h opendata.dekart.com -p 1945 -t telemetry/transport/chiK8n/1/1/+` - receive vehicle telemetry
- `mosquitto_sub -h opendata.dekart.com -p 1945 -t telemetry/route/chiK8n/1/1/+` - receive route-centric telemetry

Websocket access
----------------

For convenience, the data streams can also be accessed via web-sockets, which makes it easier to use the telemetry in web applications. Use these endpoints:

- `opendata.dekart.com:1946` - MQTT over plaintext websockets (test with http://www.hivemq.com/demos/websocket-client/)
- `https://gps.dekart.com/mqtt` - MQTT over TLS websockets (test with https://www.eclipse.org/paho/clients/js/utility/)


Feeding data into the system
============================

If vehicles in your city already have GPS trackers and you want to feed your data into Roataway:

1. Contact `support at dekart dot com`, and provide details about:
    - the city you are in
    - the name of your transportation network (e.g. "Parcul de troleibuze din Bălți", "АТБ 6, город Оргеев", "Hyperloop B44 din Drăsliceni")
    - the number of vehicles you monitor
    - the IP addresses from which your telemetry will be fed into the system
    - other details you consider relevant

2. You will be given a `peer_id` and a set of credentials you can use to connect to the MQTT server and transmit your telemetry.
3. Write the software that will form JSON payloads, as described above, and  publish them to `inbound/telemetry/<peer_id>`.

Other methods of transmission besides MQTT can be considered, if necessary.

In addition to the steps above, you will have to provide:

1. Information about your vehicles, as illustrated in `vehicles.csv`.
2. Information about your routes, as illustrated in `routes.csv`.
3. Up-to-date information about which vehicles are mapped to each of the routes, which we refer to as *vehicle-route mapping*.

Vehicle-route mapping
---------------------

This information is required to reduce visual noise on a map and display only vehicles from some routes, but not others. It can be transmitted into the system through a dedicated MQTT topic: `inbound/vehicleroutemapping/<peer_id>`. Other means of transport can be considered.

The payload should be a JSON that contains a list of 3-element entries, each consisting of:
- date in `%Y-%m-%d` format
- route name
- board number

Example: `[["2019-11-02","22","3898"],["2019-11-02","22","1289"]]`. This means that on November 2nd 2019, board 3898 is on route 22, and board 1289 is on route 22.

- You can upload mappings for several days in advance: `[["2019-11-02","22","3898"],["2019-11-03","21","3898"],["2019-11-04","30","3898"]]`. This shows how board 3898 moves from route 22 to 21 and then 30 on different days.
- You can re-upload mappings if something has changed, e.g., a vehicle was moved to another route to compensate higher demands, or if another vehicle broke down, etc.


Credits
=======

We thank [Dekart](https://dekart.com) for designing a system based on non-proprietary protocols and for opening it up to the public.

References
==========

- MQTT libraries for different [programming languages](https://www.eclipse.org/paho/downloads.php).
- Reference implementation that visualizes the [vehicles on a map](https://roataway.md/).
