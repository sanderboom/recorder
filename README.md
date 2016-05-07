# OwnTracks Recorder

![Recorder logo](assets/recorder-logo-192.png)

The _OwnTracks Recorder_ is a lightweight program for storing and accessing location data published via MQTT (or HTTP) by the [OwnTracks](http://owntracks.org) apps. It is a compiled program which is easily to install and operate even on low-end hardware, and it doesn't require an external database.

![Architecture of the Recorder](assets/ot-recorder.png)

There are two main components: the _recorder_ obtains data via MQTT subscribes or HTTP POST, stores the data in plain files and serve it via its built-in REST API, and the _ocat_ command-line utility reads stored data in a variety of formats.

We developed the _recorder_ as a one-stop solution to storing location data published by our OwnTracks apps (iOS and Android) and retrieving this data. Our previous offerings (`m2s`, `o2s`/`Pista`) also work of course, but we believe the _recorder_ is best suited to most environments. 

## `recorder`

The _recorder_ serves two purposes:

1. It subscribes to an MQTT broker and reads messages published from the OwnTracks apps, storing these in a particular fashion into what we call the _store_ which is basically a bunch of plain files on the file system. Alternatively the Recorder can listen on HTTP for OwnTracks-type JSON messages POSTed to its HTTP server.
2. It provides a Web server which serves static pages, a REST API you use to request data from the _store_, and a Websocket server. The distribution comes with a few examples of how to access the data through its HTTP interface (REST API). In particular a _table_ of last locations has been made available as well as a _live map_ which updates via the _recorder_'s Websocket interface when location publishes are received. In addition we provide maps with last points or tracks using the GeoJSON produced by the _recorder_.


## Installing

We provide a ready-to-run packages for a limited number of platforms on our [package repository](http://repo.owntracks.org/README.txt), and we provide a Docker image which bundles the Recorder and a Mosquitto broker [directly from the Docker hub](https://hub.docker.com/r/owntracks/recorderd/).

## Building from source

You will require:

* [libmosquitto](http://mosquitto.org) unless you disable MQTT during building, but see below for platform instructions
* [libCurl](http://curl.haxx.se/libcurl/)
* [lmdb](http://symas.com/mdb) (included)
* [libconfig](http://www.hyperrealm.com/libconfig/)
* Optionally [Lua](http://lua.org)
* Optionally [libsodium](https://github.com/jedisct1/libsodium) for secret-key encryption of payloads

1. Obtain and download the software, via [our Homebrew Tap](https://github.com/owntracks/homebrew-recorder) on Mac OS X, directly as a clone of the repository, or as a [tar ball](https://github.com/owntracks/recorder/releases) which you unpack.
2. Copy the included `config.mk.in` file to `config.mk` and edit that. You specify the features or tweaks you need. (The file is commented.) Pay particular attention to the installation directory and the value of the _store_ (`STORAGEDEFAULT`): that is where the recorder will store its files. `DOCROOT` is the root of the directory from which the _recorder_'s HTTP server will serve files.
3. Type `make` and watch the fun.

When _make_ finishes, you should have at least two executable programs called `ot-recorder` which is the _recorder_ proper, and `ocat`. If you want you can install these using `make install`, but this is not necessary: the programs will run from whichever directory you like if you add `--doc-root ./docroot` to the _recorder_ options.

Ensure the LMDB databases are initialized by running the following command which is safe to do, also after an upgrade. (This initialization is non-destructive -- it will not delete any data.)

```
ot-recorder --initialize
```

Unless already provided by the package you installed, we recommend you create a shell script with which you hence-force launch the _recorder_. Note that you can have it subscribe to multiple topics, and you can launch sundry instances of the recorder (e.g. for distinct brokers) as long as you ensure:

* that each instance uses a distinct `--storage`
* that each instance uses a distinct `--http-port` (or `0` if you don't wish to provide HTTP support for a particular instance)

## Getting started

The _recorder_ has, like _ocat_, a daunting number of options, most of which you will not require. Running either utility with the `-h` or `--help` switch will summarize their meanings. You can, for example launch with a specific storage directory, disable the HTTP server, change its port, etc.

If you require authentication or TLS to connect to your MQTT broker, pay attention to the `$OTR_` environment variables listed in the help.

Launch the recorder:

```
$ ./ot-recorder 'owntracks/#'
```

(In httpmode, you do not have to specify a topic.)

Publish a location from your OwnTracks app and you should see the _recorder_ receive that on the console. If you haven't disabled Geo-lookups, you'll also see the address from which the publish originated.

The location message received by the _recorder_ will be written to storage. In particular you should verify that your _storage_ directory contains:

1. a directory called `ghash/`
2. a directory called `rec/` with several subdirectories and a `.rec` file therein.
3. a directory called `last/` which contains subdirectories and a `.json` file therein.


When the recorder has received a publish or two, visit it with your favorite Web browser by pointing your browser at `http://127.0.0.1:8083` or the address / port configured with the `--http-host` and `--http-port` options respectively.

### `ot-recorder` options and variables

This section lists the most important options of the _recorder_ with their long names; check the usage (`recorder -h`) for the short versions.

`--clientid` specifies the MQTT client identifier to use upon connecting to the broker, thus overriding a constructed default. 

`--host` is the name or address of the MQTT broker and overrides `$OTR_HOST`. The default is "localhost". 

`--port` is the port number of the MQTT broker and overrides `$OTR_PORT`; it defaults to 1883. Setting this to 0 disables MQTT even if it is compiled-in.

`--user` overrides `$OTR_USER` and specifies the username to use in the MQTT connection.

`$OTR_PASS` is the password for the MQTT connection.

`$OTR_CAFILE` specifies the path to a readable PEM-formatted file containing the CA certificate chain to be used for the MQTT TLS connection. If this environment variable is set, a TLS connection is assumed (and the port number should probably be adjusted accordingly).

`--qos` specifies the MQTT QoS to use; it defaults to 2.

`--storagedir` is configured at build time and overrides `$OTR_STORAGEDIR`.

`--useretained` overrides the default of not consuming retained MQTT messages.

`--norec` disables writing of REC files, so no location history or other similar publishes are stored, and the Lua `otr_putrec()` function is not invoked even if it exists. What is stored are CARDS and PHOTOS, as well as the LAST location of a device. As such, the API's `/locations` endpoint becomes useless.

`--norevgeo` suppresses reverse geo lookups, but this means that historic data will not show addresses (e.g. with the API or with _ocat_). See below for information on Reverse Geo lookups.

`--logfacility` is the syslog facility to use (default is `LOCAL0`).

`--quiet` disables printing of messages to _stdout_.

`--initialize` creates the a structure within the storage directory and initializes the LMDB database. It is safe to use this even if such a database exists -- the database is not wiped. After initialization, _recorder_ exits.

`--label` specifies a label (default: "Recorder") to be shown in the websocket live map.

`--http-host` and `--http-port` define the listen address and port number for the API. If `--http-port` is 0, the Web server is disabled.

`--docroot` overrides the compile-time setting of the HTTP document root.

`--lua-script` specifies the path to the Lua script. If not given, Lua support is disabled.

`--precision` overrides the compiled-in default. (See "Precision" later.)

`--geokey` sets the Google API key for reverse geo lookups.  If you do more than 2500 (currently) reverse-geo requests per day, you'll need an API key for Google's geocoding service. Specify that here.

`--debug` enables a bit of additional debugging on stderr.

## The HTTP server

Some examples of what the _recorder_'s built-in HTTP server is capable of, in addition to obtaining OwnTracks app data via HTTP POST to the `/pub` endpoint.

#### Last position of a particular user

Retrieve the last position of a particular user. In addition to the values obtained in the [`location` publish](http://owntracks.org/booklet/tech/json/) from the OwnTracks device, there are a few which we return as convenience:

* `username` contains the name of the user obtained from the publish topic
* `device` contains the user's device name as obtained from the publish topic
* `topic` is the full topic to which the payload was published
* `ghash` is the geohash string which corresponds to `lat` and `lon`
* `isotst` is the ISO timestamp of the publish time (`tst`)
* `disptst` is the same but designed for displaying
* `cc` is the country code of the location point if available in the cache (see below)
* `addr` is the address of the location point if available in the cache

```
$ curl http://127.0.0.2:8083/api/0/last -d user=demo -d device=iphone
[
 {
  "tst": 1440405601,
  "acc": 10,
  "_type": "location",
  "alt": 262,
  "lon": 13.60279820860699,
  "vac": 6,
  "vel": 18,
  "lat": 51.06263391678321,
  "cog": 82,
  "tid": "NE",
  "batt": 99,
  "username": "demo",
  "device": "iphone",
  "topic": "owntracks/demo/iphone",
  "ghash": "u31dmx9",
  "isotst": "2015-08-24T08:40:01Z",
  "disptst": "2015-08-24 08:40:01",
  "cc": "DE",
  "addr": "E40, 01156 Dresden, Germany"
 }
]
```

#### Display map with points starting at a particular date

By specifying a `format` we can produce GeoJSON, say. Normally, the API retrieves the last 6 hours of data but we can extend or limit this with the `from` and `to` parameters.

```
http://127.0.0.2:8083/map/index.html?user=demo&device=iphone&format=geojson&from=2014-01-01
```

In a suitable Web browser, the result is

![GeoJSON points](assets/demo-geojson-points.png)

#### Display a track (a.k.a linestring)

If we change the `format` parameter of the previous URL to `linestring`, the result is

![GeoJSON linestring](assets/demo-geojson-linestring.png)

#### Tabular display

The _recorder_'s Web server also provides a tabular display which shows the last position of devices, their address, country, etc. Some of the columns are sortable, you can search for users/devices and click on the address to have a map opened at the device's last location.

![Table](assets/demo-table.png)

#### Live map

The _recorder_'s built-in Websocket server updates a map as it receives publishes from the OwnTracks devices. Here's an example:

![Live map](assets/demo-live-map.png)


## `ocat`

_ocat_ is a CLI query program for data stored by _recorder_: it prints data from storage in a variety of output formats:

* JSON
* GeoJSON (points)
* GeoJSON (line string)
* CSV
* GPX
* XML
* raw (the lines contained in the REC file with ISO timestamp)
* payload (basically just the payload part from RAW)

The _ocat_ utility accesses _storage_ directly — it doesn’t use the _recorder_’s REST interface. _ocat_ has a daunting number of options, some combinations of which make no sense at all.

Some example uses we consider useful:

* `ocat --list`
   show which uers are in _storage_.
* `ocat --list --user jjolie`
   show devices for the specified user
* `ocat --user jjolie --device ipad`
   print JSON data for the user's device produced during the last 6 hours.
* `ocat --last`
    print the LAST position of all users, devices. Can be combined with `--user` and `--device`.
* `ocat ... --format csv`
   produces CSV. Limit the fields you want extracted with `--fields lat,lon,cc` for example.
* `ocat ... --format xml`
   produces XML. Limit the fields you want extracted with `--fields lat,lon,cc` for example.
```xml
<?xml version='1.0' encoding='UTF-8'?>
	<?xml-stylesheet type='text/xsl' href='owntracks.xsl'?>
<owntracks>
 <point>
  <tst>1440405601</tst>
  <acc>10.000000</acc>
  <alt>262</alt>
  <lon>13.602798</lon>
  <vac>6.000000</vac>
  <vel>18</vel>
  <lat>51.062634</lat>
  <cog>82</cog>
  <tid>NE</tid>
  <batt>99</batt>
  <username>demo</username>
  <device>iphone</device>
  <topic>owntracks/demo/iphone</topic>
  <ghash>u31dmx9</ghash>
  <isotst>2015-08-24T08:40:01Z</isotst>
  <disptst>2015-08-24 08:40:01</disptst>
  <cc>DE</cc>
  <addr>E40, 01156 Dresden, Germany</addr>
 </point>

</owntracks>
```
* `ocat ... --limit 10`
   prints data  for the current month, starting now and going backwards; only 10 locations will be printed. Generally, the `--limit` option reads the storage back to front which makes no sense in some combinations.

Specifying `--fields lat,tid,lon` will request just those JSON elements from _storage_. (Note that doing so with output GPX or GEOJSON could render those formats useless if, say, `lat` is missing in the list of fields.)

The `--from` and `--to` options allow you to specify a UTC date and/or timestamp from which respectively until which data will be read. By default, the last 6 hours of data are produced. If `--from` is not specified, it therefore defaults to _now minus 6 hours_. If `--to` is not specified it defaults to _now_. Dates and times must be specified as strings, and the following formats are recognized:

```
%Y-%m-%dT%H:%M:%S
%Y-%m-%dT%H:%M
%Y-%m-%dT%H
%Y-%m-%d
%Y-%m
```

The `--limit` option limits the output to the last specified number of records. This is a bit of an "expensive" operation because we search the `.rec` files backwards (i.e. from end to beginning). When using `--limit` the 6 hours mentioned earlier do not apply.

## `ocat` examples

The _recorder_ has been running for a while, and the OwnTracks apps have published data. Let us have a look at some of this data.

#### List users and devices

We obtain a list of users from the _store_:

```
$ ocat --list
{
 "results": [
  "demo"
 ]
}
```

From which devices has user _demo_ published data?

```
$ ocat --list --user demo
{
 "results": [
  "iphone"
 ]
}
```

#### Show the last position reported by a user

Where was _demo_'s _iphone_ last seen? (Omit `--user` and `--device` to get LAST for all users and devices.)

```
$ ocat --last --user demo --device iphone
[
 {
  "tst": 1440405601,
  "acc": 10,
  "_type": "location",
  "alt": 262,
  "lon": 13.60279820860699,
  "vac": 6,
  "vel": 18,
  "lat": 51.06263391678321,
  "cog": 82,
  "tid": "NE",
  "batt": 99,
  "username": "demo",
  "device": "iphone",
  "topic": "owntracks/demo/iphone",
  "ghash": "u31dmx9",
  "isotst": "2015-08-24T08:40:01Z",
  "disptst": "2015-08-24 08:40:01",
  "cc": "DE",
  "addr": "E40, 01156 Dresden, Germany"
 }
]
```

Several things worth mentioning:

* The returned data structure is an array of JSON objects; had we omitted specifying a particular device or even a particular user we would have obtained the last position of all this user's devices or all users' devices respectively.
* If you are familiar with the [JSON data reported by the OwnTracks apps](http://owntracks.org/booklet/tech/json/) you'll notice that this JSON contains more information: this is provided on the fly by _ocat_ and the REST API, e.g. from the reverse-geo cache the _recorder_ maintains.

#### What were the last 4 positions reported?

We can limit the number of returned elements: Let's do this as CSV, and limit the fields we are given:

```
$ ocat --user demo --device iphone --limit 4 --format csv --fields isotst,vel,addr
isotst,vel,addr
2015-08-24T08:40:01Z,18,"E40, 01156 Dresden, Germany"
2015-08-24T08:35:01Z,40,"E40, 01723 Wilsdruff, Germany"
2015-08-24T08:30:00Z,50,"A14, 01683 Nossen, Germany"
2015-08-24T08:24:59Z,40,"A14, 04741 Roßwein, Germany"
```


## Design decisions

We took a number of decisions when designing the _recorder_ and its utilities:

* Flat files. The filesystem is the database. Period. That's were everything is stored. It makes incremental backups, purging old data, manipulation via the Unix toolset easy. (Admittedly, for fast geo-lookups we employ LMDB as a cache, but the final word is in the filesystem.) We considered all manner of databases and decided to keep this as simple and lightweight as possible. You can however have the _recorder_ send data to a database of your choosing, in addition to the file system it uses, by utilizing our embedded Lua hook.
* We wanted to store received data in the format it's published in. As this format is JSON, we store this raw payload in the `.rec` files. If we add an attribute to the JSON published by our apps, you have it right there. There's one slight exception: the monthly logs (the `.rec` files) have a leading timestamp and a relative topic; see below. (In the particular case of the OwnTracks firmware for Greenwich devices which can publish in CSV mode, we convert the CSV into OwnTracks JSON for storage.)
* File names are lower case. A user called `JaNe` with a device named `myPHONe` will be found in a file named `jane/myphone`.
* All times are UTC (a.k.a. Zulu or GMT). We got sick and tired of converting stuff back and forth. It is up to the consumer of the data to convert to localtime if need be.
* The _recorder_ does not provide authentication or authorization. Nothing at all. Zilch. Nada. Think about this before making it available on a publicly-accessible IP address. Or rather: don't think about it; just don't do it. You can of course place a HTTP proxy in front of the `recorder` to control access to it. Or use views (see below).
* `ocat`, the _cat_ program for the _recorder_ uses the same back-end which is used by the API though it accesses it directly (i.e. without resorting to HTTP).
* The _recorder_ supports 3-level MQTT topics only, in the typical OwnTracks format: `"owntracks/<username>/<devicename>"`, optionally with a leading slash. (The first part of the topic need not be "owntracks".) Publishes via HTTP POST construct a ficticious topic internally using the provided user (`u`) and device (`d`) parameters.

## Storage

As mentioned earlier, data is stored in files, and these files are relative to `STORAGEDIR` (compiled into the programs or specified as an option). In particular, the following directory structure can exist, whereby directories are created as needed by the _recorder_:

* `cards/`, optional, may contains user cards. This card is then stored here and used with, e.g., `ocat --last` to show a user's name and optional avatar. User cards are typically stored in a subdirectory called _username_, and therein a JSON file _username_.json. When reading cards, the recorder will first attempt to open _username_/_device_/_username_.json and then _username_/_username_.json.
* `config/`, optional, contains the JSON of a [device configuration](http://owntracks.org/booklet/features/remoteconfig/) (`.otrc`)  which was requested remotely via a [dump command](http://owntracks.org/booklet/tech/json/#_typecmd). Note that this will contain sensitive data. You can use this `.otrc` file to restore the OwnTracks configuration on your device by copying to the device and opening it in OwnTracks.
* `ghash/`, unless disabled, reverse Geo data (using a Google service) is collected into an LMDB database located in this directory. This LMDB database also contains named databases which are used by your optional Lua hooks, as well as a `topic2tid` database which can be used for TID re-mapping.
* `last/` contains the last location published by devices. E.g. Jane's last publish from her iPhone would be in `last/jjolie/iphone/jjolie-iphone.json`. The JSON payload contained therein is enhanced with the fields `user`, `device`, `topic`, and `ghash`. If a device's `last/` directory contains a file called `extra.json` (i.e. matching the example, this would be `last/jjolie/iphone/extra.json`), the content of this file is merged into the existing JSON for this user and returned by the API. Note, that you cannot overwrite existing values. So, an `extra.json` containing `{ "tst" : 11 }` will do nothing because the `tst` element we obtain from location data overrules, but adding `{ "beverage" : "water" }` will do what you want. If _recorder_ is built with support for our Greenwich firmware, this directory might contain `batt.json`, `ext.json`, and/or `status.json` each of which hold an array of the last 100 reports for internal battery voltage, external voltage, and status respectively. These values are returned via the API in the LAST object. A file `http.json` which should contain either a single JSON object or an array of JSON objects is returned to clients in HTTPmode.
* `monitor` a file which contains a timestamp and the last received topic (see Monitoring below).
* `msg/` contains messages received by the Messaging system.
* `photos/` optional; contains the binary photos from a _card_.
* `rec/` the recorder data proper. One subdirectory per user, one subdirectory therein per device. Data files are named `YYYY-MM.rec` (e.g. `2015-08.rec` for the data accumulated during the month of August 2015.
* `waypoints/` contains a directory per user and device. Therein are individual files named by a timestamp with the JSON payload of published (i.e. shared) waypoints. The file names are timestamps because the `tst` of a waypoint is its key. If a user publishes all waypoints from a device (Publish Waypoints), the payload is stored in this directory as `username-device.otrw`. (Note, that this is the JSON [waypoints import format](http://owntracks.org/booklet/tech/json/#_typewaypoints).) You can use this `.otrw` file to restore the waypoints on your device by copying to the device and opening it in OwnTracks.

You should definitely **not** modify or touch these files: they remain under the control of the _recorder_. You can of course, remove old `.rec` files if they consume too much space.

## Configuration file

The recorder attempts to read its startup configuration from a configuration file; the path to this is compiled into the Recorder (typically `/etc/defaults/ot-recorder`, and `ocat -v` will display the compiled-in default). The format of this file approximates that of a shell script with variables to be exported (the intention is so that it can be sourced by a shell script). Lines beginning with an octothorp (`#`) are ignored as are blank lines. Configuration settings proper are set as follows (note that some older versions of _libconfig_ require a trailing semicolon (`;`) at the end of a variable assignment):

```
OTR_STORAGEDIR="/var/spool/owntracks/recorder/store"
```

The following configuration settings may be applied (a `Y` in column `$` means an environment variable of the same name overrides a setting in the config file):

| Variable              |  $    | Default       | Usage
| --------------------- | :---  | :------------ | ---------------
| `OTR_STORAGEDIR`      |  Y    | compiled in   | Pathname to the storage directory
| `OTR_HOST`            |  Y    | `localhost`   | MQTT hostname/address to connect to
| `OTR_PORT`            |  Y    | `1883`        | MQTT port number to connect to
| `OTR_USER`            |  Y    |               | MQTT username
| `OTR_PASS`            |  Y    |               | MQTT password
| `OTR_QOS`             |       | `2`           | MQTT QoS
| `OTR_CLIENTID`        |       | hostname+pid  | MQTT ClientID (override with -i)
| `OTR_HTTPHOST`        |       | `localhost`   | Address for the HTTP module to bind to
| `OTR_HTTPPORT`        |       | `8083`        | Port number of the HTTP module to bind to
| `OTR_HTTPLOGDIR`      |       |               | Directory in which to store access.log. Override with --http-logdir
| `OTR_LUASCRIPT`       |       |               | Path to the Lua script
| `OTR_PRECISION`       |       | `7`           | Reverse-geo precision
| `OTR_GEOKEY`          |       |               | API key for reverse-geo lookups
| `OTR_TOPICS`          |       |               | String containing a space-separated list of topics to subscribe to for MQTT (overriden by command-line arguments)
| `OTR_CAFILE`          |  Y    |               | Path to PEM-encoded CA certificate file for MQTT


Note that options passed to `ot-recorder` override both configuration file settings and environment variables.


## Reverse Geo

If not disabled with option `--norevgeo`, the _recorder_ will attempt to perform a reverse-geo lookup on the location coordinates it obtains and store them in an LMDB database. If a lookup is not possible, for example because you're over quota, the service isn't available, etc., _recorder_ keeps tracks of the coordinates which could *not* be resolved in a file named `missing`:

```
$ cat store/ghash/missing
u0tfsr3 48.292223 8.274535
u0m97hc 46.652733 7.868803
...
```

This can be used to subsequently obtain missed lookups.

We recommend you keep reverse-geo lookups enabled, this data (country code `cc`, and the locations address `addr`) is used by the example Web apps provided by the _recorder_ to show where a particular device is. In addition, this cached data is used the the API (also _ocat_) when printing location data.

### Precision

The precision with which reverse-geo lookups are performed is controlled with the `--precison` option to _recorder_ (and with the `--precision` option to _ocat_ when you query for data). The default precision is compiled into the code (from `config.mk`). The higher the number, the more frequently lookups are performed; conversely, the lower the number, the fewer lookups are performed. For example, a precision of 1 means that points within an area of approximately 5000 km^2 would resolve to a single address, whereas a precision of 7 means that points within an area of approximately 150 m^2 resolve to one address. The _recorder_ obtains a location publish, extracts the latitude and longitude, and then calculates the [geohash](https://en.wikipedia.org/wiki/Geohash) string and truncates it to _precision_. If the calculated geohash string can be found in our local LMDB cache, we consider the point cached; otherwise an actual reverse geo lookup (via HTTP) is performed and the result is cached in LMDB at the key of the geohash.

As an example, let's assume Jane's device is at position (lat, lon) `48.879840, 2.323522`, which resolves to a geohash string of length 7 `u09whf7`. We can [visualize this](http://www.movable-type.co.uk/scripts/geohash.html) and show what this looks like. (See also: [visualizing geohash](http://www.bigdatamodeling.org/2013/01/intuitive-geohash.html).)

![geohash7](assets/geohash-7.png)

Every location publish outside that very small blue square would mean another lookup. If, however, we lower the precision to, say, 5, a much larger area is covered

![geohash5](assets/geohash-5.png)

and a precision of 2 would mean that a very large part of France resolves to a single address:

![geohash2](assets/geohash-2.png)

The bottom line: if you run the _recorder_ with just a few devices and want to know quite exactly where you've been, use a high precision (7 is probably good). If you, on the other hand, run _recorder_ with many devices and are only interested in where a device was approximately, lower the precision; this also has the effect that fewer reverse-geo lookups will be performed in the Google infrastructure. (Also: respect their quotas!)

### The geo cache

As hinted to above, the address data obtained through a reverse-geo lookup is stored in an embedded LMDB database, the content of which we can look at with

```
$ ocat --dump
u09whf7 {"cc":"FR","addr":"1 Rue de Saint-Pétersbourg, 75008 Paris, France","tst":1445435622,"locality":"Paris"}
u09ey1r {"cc":"FR","addr":"D83, 91590 La Ferté-Alais, France","tst":1445435679,"locality":"La Ferté-Alais"}
```

The key to this data is the geohash string (here with an example of precision 2).

## Monitoring

In order to monitor the _recorder_, whenever an MQTT message is received, a `monitor` file located relative to STORAGEDEFAULT is maintained. It contains a single line of text: the epoch timestamp and the last received topic separated from each other by a space.

```
1439738692 owntracks/jjolie/ipad
```


If _recorder_ is built with `WITH_PING` (default), a location publish to `owntracks/ping/ping` (i.e. username is `ping` and device is `ping`) can be  used to round-trip-test the recorder. For this particular username/device combination, _recorder_ will store LAST position, but it will not keep a `.REC` file for it. This can be used to verify, say, via your favorite monitoring system, that the _recorder_ is still operational.

After sending a _pingping_, you can query the REST interface to determine the difference in time. The `contrib/` directory has an example Python program (`ot-ping.py`) which you can adapt as needed for use by Icinga or Nagios.

```
OK ot-recorder pingping at http://127.0.0.1:8085: 0 seconds difference
```

## HTTP server

The _recorder_ has a built-in HTTP server with which it servers static files from either the compiled-in default `DOCROOT` directory or that specified at run-time with the `--doc-root` option. Furthermore, it serves JSON data from the API end-point at `/api/0/` and it has a built-in Websocket server for the live map.

The API basically serves the same data as _ocat_ is able to produce.

### API

The _recorder_'s API provides most of the functions that are surfaced by _ocat_. GET and POST requests are supported, and if a username and device are needed, these can be passed in via `X-Limit-User` and `X-Limit-Device` headers alternatively to GET or POST parameters. (From and To dates may also be specified as `X-Limit-From` and `X-Limit-To`
respectively.)

The API endpoint is at `/api/0` and is followed by the verb.

#### `monitor`

Returns the content of the `monitor` file as plain text.

```
curl 'http://127.0.0.1:8083/api/0/monitor'
1441962082 owntracks/jjolie/phone
```

#### `last`

Returns a list of last users' positions. (Can be limited by _user_, _device_, and _fields_, a comma-separated list of fields which should be returned instead of the default of all fields.)

```
curl http://127.0.0.1:8083/api/0/last [-d user=jjolie [-d device=phone]]
```

```
curl 'http://127.0.0.1:8083/api/0/last?fields=tst,tid,addr,topic,isotst'
```

#### `list`

List users. If _user_ is specified, lists that user's devices. If both _user_ and _device_ are specified, lists that device's `.rec` files.

#### `locations`

Here comes the actual data. This lists users' locations and requires both _user_ and _device_. Output format is JSON unless a different _format_ is given (`csv`, `json`, `geojson`, `xml`, and `linestring` are supported).

In order to limit the number of records returned, use _limit_ which causes a reverse search through the `.rec` files; this can be used to find the last N positions.

Date/time ranges may be specified as _from_ and _to_ with dates/times specified as described for _ocat_ above.

```
curl http://127.0.0.1:8083/api/0/locations -d user=jpm -d device=5s
curl http://127.0.0.1:8083/api/0/locations -d user=jpm -d device=5s -d limit=1
curl http://127.0.0.1:8083/api/0/locations -d user=jpm -d device=5s -d format=geojson
curl http://127.0.0.1:8083/api/0/locations -d user=jpm -d device=5s -d from=2014-08-03
curl 'http://127.0.0.1:8083/api/0/locations?from=2015-09-01&user=jpm&device=5s&fields=tst,tid,addr,isotst'
```

#### `q`

Query the geo cache for a particular _lat_ and _lon_.

```
curl 'http://127.0.0.1:8083/api/0/q?lat=48.85833&lon=2.295'
{
 "cc": "FR",
 "addr": "9 Avenue Anatole France, 75007 Paris, France",
 "tst": 1441984405
}
```

The reported timestamp was the time at which this cache entry was made. Note that this interface queries only -- it does not populate the cache.

#### `photo`

Requires GET method and _user_, and will return the `image/png` 40x40px photograph of a user if available in `STORAGEDIR/photos/` or a transparent 40x40png with a black border otherwise.

#### `kill`

If support for this is compiled in, this API endpoint allows a client to remove data from _storage_. (Warning: *any* client can do this, as there is no authentication/authorization in the _recorder_!)

```
curl 'http://127.0.0.1:8083/api/0/kill?user=ngin&device=ojo'

{
 "path": "s0/rec/ngin/ojo",
 "status": "OK",
 "last": "s0/last/ngin/ojo/ngin-ojo.json",
 "killed": [
  "2015-09.rec",
 ]
}
```
The response contains a list of removed `.rec` files, and file system operations are logged to syslog.

#### `version`

Returns a JSON object which contains the Recorder's version string, such as

```json
{ "version": "0.4.7" }
```

## Lua hooks

If _recorder_ is compiled with Lua support, a Lua script you provide is launched at startup. Lua is _a powerful, fast, lightweight, embeddable scripting language_. You can use this to process location publishes in any way you desire: your imagination (and Lua-scripting knowhow) set the limits. Some examples:

* insert publishes into a database of your choice
* switch on the coffee machine when your OwnTracks device reports you're entering home (but see also [mqttwarn](http://jpmens.net/2014/02/17/introducing-mqttwarn-a-pluggable-mqtt-notifier/))
* write a file with data in a format of your choice (see `etc/example.lua`)

Run the _recorder_ with the path to your Lua script specified in its `--lua-script` option (there is no default). If the script cannot be loaded (e.g. because it cannot be read or contains syntax errors), the _recorder_ unloads Lua and continues *without* your script.

If the Lua script can be loaded, it is automatically provided with a table variable called `otr` which contains the following members:

* `otr.version` is a read-only string with the _recorder_ version (example: `"0.3.2"`)
* `otr.log(s)` is a function which takes a string `s` which is logged to syslog at the _recorder_'s facility and log level INFO.
* `otr.strftime(fmt, t)` is a function which takes a format string `fmt` (see `strftime(3)`) and an integer number of seconds `t` and returns a string with the formatted UTC time. If `t` is 0 or negative, the current system time is used.
* `otr.putdb(key, value)` is a function which takes two strings `k` and `v` and stores them in the named LMDB database called `luadb`. This can be viewed with
* `otr.getdb(key)` is a function which takes a single string `key` and returns the database value associated with that key or `nil` if the key isn't stored.

```
ocat --dump=luadb
```

Your Lua script *must* provide the following functions:

### `otr_init`

This is invoked at start of _recorder_. If the function returns a non-zero value, _recorder_ unloads Lua and disables its processing; i.e. the `hook()` will *not* be invoked on location publishes.

### `otr_exit`

This is invoked when the _recorder_ stops, which it doesn't really do unless you CTRL-C it or send it a SIGTERM signal.


### `otr_hook`

This function is invoked at every location publish processed by the _recorder_. Your function is passed three arguments:

1. _topic_ is the topic published to (e.g. `owntracks/jane/phone`)
2. _type_ is the type of MQTT message. This is the `_type` in our JSON messages (e.g. `location`, `cmd`, `transition`, ...) or `"unknown"`.
3. _location_ is a [Lua table](http://www.lua.org/pil/2.5.html) (associative array) with all the elements obtained in the JSON message. In the case of _type_ being `location`, we also add country code (`cc`) and the location's address (`addr`) unless reverse-geo lookups have been disabled in _recorder_.

Assume the following small example Lua script in `example.lua`:

```lua
local file

function otr_init()
	otr.log("example.lua starting; writing to /tmp/lua.out")
	file = io.open("/tmp/lua.out", "a")
	file:write("written by OwnTracks Recorder version " .. otr.version .. "\n")
end

function otr_hook(topic, _type, data)
	local timestr = otr.strftime("It is %T in the year %Y", 0)
	print("L: " .. topic .. " -> " .. _type)
	file:write(timestr .. " " .. topic .. " lat=" .. data['lat'] .. data['addr'] .. "\n")
end

function otr_exit()
end
```

When _recorder_ is launched with `--lua-script example.lua` it invokes `otr_init()` which opens a file. Then, for each location received, it calls `otr_hook()` which updates the file.

Assuming an OwnTracks device publishes this payload

```json
{"cog":-1,"batt":-1,"lon":2.29513,"acc":5,"vel":-1,"vac":-1,"lat":48.85833,"t":"u","tst":1441984413,"alt":0,"_type":"location","tid":"JJ"}
```

the file `/tmp/lua.out` would contain

```txt
written by OwnTracks Recorder version 0.3.0
It is 14:10:01 in the year 2015 owntracks/jane/phone lat=48.858339 Avenue Anatole France, 75007 Paris, France
```

### `otr_putrec`

An optional function you provide is called `otr_putrec(u, d, s)`. If it exists,
it is called with the current user in `u`, the device in `d` and the payload
(which for OwnTracks apps is JSON but for, eg Greenwich devices might not be) in the string `s`. If your function returns a
non-zero value, the _recorder_ will *not* write the REC file for this publish.

### `otr_httpobject`

An optional function you provide is called `otr_httpobject(u, d, t, data)` where `u` is the username used by the client (`?u=`), `d` is the device name (`&d=` in the URI), `t` is the OwnTracks JSON `_type` and `data` a Lua table built from the OwnTracks JSON payload of `_type`. If it exists, this function is called whenever a POST is received in httpmode and the Recorder is gathering data to return to the client app. The function *must* return a Lua table containing any number of string, number, or boolean values which are converted to a JSON object and appended to the JSON array returned to the client. An [example](etc/example.lua) shows how, say, a transition event can be used to open the Featured content tab in the app.

### Hooklets

After running `otr_hook()`, the _recorder_ attempts to invoke a Lua function for each of the elements in the extended JSON. If, say, your Lua script contains a function called `hooklet_lat`, it will be invoked every time a `lat` is received as part of the JSON payload. Similarly with `hooklet_addr`, `hooklet_cc`, `hooklet_tst`, etc. These _hooklets_ are invoked with the same parameters as `otr_hook()`.

You define a hooklet function only if you're interested in expressly triggering on a particular JSON element.


### Environment

The following environment variables control _ocat_'s behaviour:

* `OCAT_FORMAT` can be set to the preferred output format. If unset, JSON is used. The `--format` option overrides this setting.
* `OCAT_USERNAME` can be set to the preferred username. The `--user` option overrides this environment variable.
* `OCAT_DEVICE` can be set to the preferred device name. The `--device` option overrides this environment variable.

### nginx

Running the _recorder_ protected by an _nginx_ or _Apache_ server is possible and is the only recommended method if you want to server data behind _localhost_. This snippet shows how to do it, but you would also add authentication to that.

```
server {
    listen       8080;
    server_name  192.168.1.130;

    location / {
        root   html;
        index  index.html index.htm;
    }

    # Proxy and upgrade Websocket connection
    location /otr/ws {
    	rewrite ^/otr/(.*)	/$1 break;
    	proxy_pass		http://127.0.0.1:8084;
    	proxy_http_version	1.1;
    	proxy_set_header	Upgrade $http_upgrade;
    	proxy_set_header	Connection "upgrade";
    	proxy_set_header	Host $host;
    	proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /otr/ {
    	proxy_pass		http://127.0.0.1:8084/;
    	proxy_http_version	1.1;
    	proxy_set_header	Host $host;
    	proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Apache

Assuming you want to use Apache as a reverse proxy to the recorder, the following
may get you started. This will hand URIs which begin with `/otr/` to the _Recorder_.

```

# Websocket URL endpoint
# a2enmod proxy_wstunnel
ProxyPass        /otr/ws        ws://127.0.0.1:8083/ws keepalive=on retry=60
ProxyPassReverse /otr/ws        ws://127.0.0.1:8083/ws keepalive=on

# Static files
ProxyPass /otr                  http://127.0.0.1:8083/
ProxyPassReverse /otr           http://127.0.0.1:8083/
```

## Views

A view is a sort of sandboxed look at data provided by the Recorder. Assume you host several devices, be they your own or those of some of your friends, and assume you want to allow somebody else to see where you are or have been during a specific time frame: with the Recorder's default Web server you cannot limit a visitor to see specific data only; once they reach the Recorder's Web interface, they have access to all your data. (We warned you about that earlier.) Using a HTTP proxy, you can provide an insight into certain portions of your data only.

You configure a view by creating a small JSON file of an arbitrary name which defines which user / device combination of data the view should display. Say you are recording data for `owntracks/jjolie/phone`, the _user_ would be `jjolie` and the _device_ is `phone`. You can also create a specific HTML page for this view or just use the default `vmap.html` we provide.

The view then provides three URLs:

* `views/viewname` can serve a HTML or text page
* `views/viewname?lastpos=1` serves a JSON array of objects with the last position recorded
* `views/viewname?geodata=1` serves a GeoJSON object containing recorded track data

Suppose Jane wishes to have her acqaintances see where she is whilst on vacation. Jane knows she'll be en-route between 2015-06-29 and 2015-07-15. She creates a file called, say, `loire.json` in the `views/` directory of the Recorder's document root:

```json
{
  "user"  : "jjolie",
  "device": "phone",
  "page"  : "vmap.html",
  "from"  : "2015-06-29",
  "to"    : "2015-07-15"
}
```

Jane's friends can now visit the URL `/view/loire` (note the missing `.json` extension) to be served a map showing Jane's progress along the Loire valley (if that is where she's actually travelling through). Jane can keep that view up even after she returns because the view will not serve data after the 15th of July, in other words, her location at any other time before or after the _from_ / _to_ dates is hidden.

![Jane's vacation](assets/view-map.png)

### view JSON

The JSON in the view file (called `view.json` here) contains mandatory and optional elements:

| element         |  mandatory  | meaning                               |
| ----------------| :---------: | ------------------------------------- |
| user            |      Y      | username for data (from topic owntracks/user/device       |
| device          |      Y      | device for data (from topic owntracks/user/device       |
| page            |      Y      | HTML page to be loaded from `docroot/views/` for this view |
| from            |      N      | `from` timestamp for data, defaults to now - 6H |
| to              |      N      | `to` timestamp for data, defaults to now  |
| hours           |      N      | number of past hours (from "now") for which to show data; use instead of `from` and `to` 
| auth            |      N      | array of digest authentication tokens described below |
| label           |      N      | text to use in popup of default `vmap.html` instead of user/device |
| zoom            |      N      | zoom level for map used in `vmap.html`, defaults to 9 |
| *               |      N      | any other element is copied into the data returned |



The _page_ is a single HTML file which must be located in the `views/` directory of the Recorder's document root. Trivial (primitive actually) text substitution is done for the following two tokens:

* `@@@LASTPOS@@@` is converted to a URI on which the Recorder will serve the last position data
* `@@@GEO@@@` is converted to a URI on which the Recorder will serve GeoJSON data from its storage.

The default _page_ we provide is called `vmap.html`; by default it refreshes the last position every 60 seconds, and clicking on _Load track_ loads the GeoJSON track for the time frame specified by `from` and `to`. 

![Jane's vacation track](assets/view-track.png)

A little bit more complex view would look like this:

```json
{
  "config": {
    "port": 9001,
    "pathname": "/tmp/somewhere"
  },
  "zoom": 7,
  "label": "Jane's Loire vacation",
  "to": "2015-07-15",
  "from": "2015-06-29",
  "device": "phone",
  "user": "jjolie",
  "page": "vmap.html"
}
```

All JSON elements are copied into the _lastpos_ data which is returned to the caller. Using the above view configuration, a user requesting `http://localhost:8083/view/loire?lastpos=1` would obtain

```json
{
 "data": [
  {
   "_type": "location",
   "cc": "FR",
   "lon": -1.564214,
   "lat": 47.217871,
   "alt": 35,
   "vel": 0,
   "t": "L",
   "cog": 0,
   "tid": "K2",
   "tst": 1436895718,
   "ghash": "gbqus7u",
   "addr": "Maison d'arrêt, 9 Rue Descartes, 44000 Nantes, France",
   "locality": "Nantes",
   "isorcv": "2015-07-14T19:41:58Z",
   "isotst": "2015-07-14T17:41:58Z",
   "disptst": "2015-07-14 17:41:58",
   "page": "vmap.html",
   "user": "jjolie",
   "device": "phone",
   "from": "2015-06-29",
   "to": "2015-07-15",
   "label": "Jane's Loire vacation",
   "zoom": 7,
   "pathname": "/tmp/somewhere",
   "port": 9001
  }
 ]
}
```

Note how `pathname` and `port` have been copied into the object. These values can be used by the _page_ served in the view.

### Authentication

If `view.json` contains an element called `auth`, it is assumed to be an array of strings, each of which are a 32-character [Digest authentication](https://en.wikipedia.org/wiki/Digest_access_authentication) SHA1 strings for the realm `owntracks-recorder`, for example:

```json
"auth" : [ "225544f9acf99d18a8880c5ce844f303", "ba69e267302a7ef98e0862b9aae68cab" ]
```

Each user/password digest combination will be able to access the view.

You create these strings with, say, the _htdigest_ program or `contrib/new-view-auth.py`:

```bash
htdigest -c /tmp/dd owntracks-recorder jjolie
Adding password for jjolie in realm owntracks-recorder.
New password:
Re-type new password:

cat /tmp/dd
jjolie:owntracks-recorder:225544f9acf99d18a8880c5ce844f303
```

In the above example, you copy the 32-character digest into your `view.json`, whereas in the following example, we create a template for you which you copy into your view.

```bash
./new-view-auth.py jjolie
Enter password for user jjolie:
Re-enter password:
"auth" : [ "225544f9acf99d18a8880c5ce844f303" ]
```


### HTTP proxy

We recommend you have the Recorder listening to a loopback interface (e.g. 127.0.0.1) as it does by default, and set up a reverse proxy to its views. Using _nginx_ the following configuration shows how we proxy the `view/` and the required `static/` URIs into the Recorder:

```
# OwnTracks Recorder Views
location /owntracks/view/ {
     proxy_buffering         off;            # Chrome
     proxy_pass              http://127.0.0.1:8085/view/;
     proxy_http_version      1.1;
     proxy_set_header        Host $host;
     proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
}
location /owntracks/static/ {
     proxy_pass              http://127.0.0.1:8085/static/;
     proxy_http_version      1.1;
     proxy_set_header        Host $host;
     proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

You would then visit `http://example.com/owntracks/view/loire` to see the `loire` view, assuming `example.com` is your proxy.

## HTTP mode

If enabled at compile time (`WITH_HTTP`), the Recorder will accept OwnTracks-type JSON payloads via HTTP at the URL endpoint `/pub&u=username&d=device`. You specify the username with the `u` parameter and the device name with the `d` parameter. (Alternatively you can provide `X-Limit-U` and `X-Limit-D` as headers with the username and device name respectively.) If unspecified, the username defaults to `owntracks` and the device to `phone`. For example:

```
curl --data "${payload}" 'http://127.0.0.1:8085/pub?u=jane&d=3s'
curl -H 'X-Limit-U: jane' -H 'X-Limit-D: 3s' --data "${payload}" 'http://127.0.0.1:8085/pub'
```

The content of the request is used by the Recorder as though it had arrived as an MQTT message; Lua hooks and Websocket pushes are handled accordingly.

If the Recorder is compiled without specifying `WITH_MQTT` at build time, support for MQTT is disabled completely.

### Friends in HTTP mode

When a device posts a location request in HTTP mode, the endpoint may return a JSON array of OwnTracks objects of which `_type`s `cmd`, `location` and `card` may be supported by the device. This allows the device to see, say, friends. The Recorder has built-in support for this with the named "friends" lmdb database.
Assuming the following content of the friends database

```
$ ocat -S JP --dump=friends
jane-phone ["john/android", "jane/phone"]
```

when user `jane` and device `phone` POST a new location via HTTP, the Recorder will present John's data as the following payload to the device, assuming John has a card and last published this location:

```json
[
 {
  "_type": "card",
  "tid": "JA",
  "face": "/9j/4AAQSkZJR...",
  "name": "John Doe"
 },
 {
  "_type": "location",
  "tid": "JA",
  "lat": 48.95833,
  "lon": 2.39523,
  "tst": 1456212791
 }
]
```

Note, that Jane's user/device tuple should also be returned in order to display Jane on the map or list of friends in the apps.

### Authentication

In HTTP mode, the Recorder provides no form of authentication; anybody who "stumbles" over the correct endpoint will be able to post location data to your Recorder! You do not want this to happen.

Install, say, an _nginx_ proxy before it and ensure it's configured for HTTP basic authentication:

```
# - Recorder PUB -----------------------------------------------------------
location /owntracks/pub {
    auth_basic              "OwnTracks pub";
    auth_basic_user_file    /usr/local/etc/nginx/owntracks.htpasswd;
    proxy_pass              http://127.0.0.1:8083/pub;
    proxy_http_version      1.1;
    proxy_set_header        Host $host;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;

    # Optionally force Recorder to use username from Basic
    # authentication user. Whether or not client sets
    # X-Limit-U and/or uses ?u= parameter, the user will
    # be set to $remote_user.
    proxy_set_header        X-Limit-U $remote_user;
}
```

## Advanced topics

### The LMDB database

`ocat --load` and `ocat --dump` can be use to load and dump the lmdb database respectively. There is some support for loading/dumping named databases using `--load=xx` or `--dump=xx` to specify the name. Use the mdb utilities to actually perform backups of these. _load_ expects key/value strings in pairs, separated by exactly one space. If the value is the string `DELETE`, the key is deleted from the database, which allows us to, say, remove a whole bunch of geohash prefixes in one go (but be careful doing this):

```bash
ocat --dump |
    grep xxyz |
    awk '{printf "%s DELETE\n", $1; }' |
    ocat --load
```

#### `topic2tid`

This named lmdb database is keyed on topic name (`owntracks/jane/phone`). If the topic of an incoming message is found in the database, the `tid` member in the JSON payload is replaced by the string value of this key.

#### `keys`

If the _recorder_ was built with encryption support (see below), this named database contains the secret decryption keys for users/device pairs. The LMDB key is the username followed by a dash followed by the device name, all lower case, with spaces translated to a single dash. For example, if user Jjolie with device iPhone needs a secret entered, the database key will be `jjolie-iphone`. This can be entered into the database as follows:

```bash
echo "jjolie-iphone s3cr1t" | ocat --load=keys
```

Beware: these secret keys are stored in plain text so the database must be protected!

#### `friends`

For http mode, the `friends` named LMDB database contains lists of "friends" on a per user-device key. The key's value must be a valid JSON array of strings, each in the form `"user:device"` or `"user/device"` which indicate which locations a particular user may see. For example, when a user called `jane` on device `phone` publishes in http mode and she should be permitted to see where `john` / `android` is, we add the following key/value to the friends named database:

```bash
ocat --load=friends <<EOF
jane-phone [ "john/android" ]
EOF
```

The user/device separator in the array's strings may be a slash (`/`), a dash (`-`), or a colon (`:`).

## Encryption (*experimental!*)

If compiled with `WITH_ENCRYPT` support (this is the default in our packages), the recorder will handle messages from OwnTracks [devices which support payload encryption](http://owntracks.org/booklet/features/encrypt/). Each user / device requires a secret key which is configured on the device and which must be configured on the Recorder host in order for the Recorder to be able to decrypt the payloads.

Upon successful decryption, the Recorder processes the original (device-transmitted) JSON and stores the result in plain (i.e. un-encrypted) form in the store.



## Prerequisites for building

You need a current version of the Mosquitto library (and you probably require the Mosquitto broker as well for OwnTracks). We strongly recommend installing Mosquitto either from [source](http://mosquitto.org/download/) or from a [binary package](http://mosquitto.org/download/), both of which are provided by the [Mosquitto project](http://mosquitto.org/). In particular, older or LTS OS versions profit from this.

### Debian

```
apt-get install build-essential linux-headers-$(uname -r) libcurl4-openssl-dev libmosquitto-dev liblua5.2-dev libsodium-dev
```

### Centos 7

```
yum groupinstall 'Development Tools'
yum install libmosquitto-devel libcurl-devel lua-devel libsodium-devel libconfig-devel
```

libsodium is in epel-stable

### Ubuntu

```
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update
sudo apt-get install libmosquitto-dev
sudo apt-get install libcurl3 libcurl4-openssl-dev
```

[![Build Status](https://travis-ci.org/owntracks/recorder.svg?branch=master)](https://travis-ci.org/owntracks/recorder)

## Packages

We create packages for releases for a few distributions. Please note that these packages depend on libmosquitto1 from the [Mosquitto project](http://mosquitto.org/downloads).

Binaries (`ocat`, `ot-recorder`) from these packages run setuid to user `owntracks` so that they work for all users of the system. Note that, say, certificate files you provide must therefore also be readable by the user `owntracks`.

### Installing on Centos 7

```
curl -o /etc/yum.repos.d/mosquitto.repo http://download.opensuse.org/repositories/home:/oojah:/mqtt/CentOS_CentOS-7/home:oojah:mqtt.repo

curl -o /etc/yum.repos.d/owntracks.repo http://repo.owntracks.org/centos/owntracks.repo

yum install ot-recorder
```

### Installing on Raspian (wheezy)

```
wget http://repo.owntracks.org/repo.owntracks.org.gpg.key
apt-key add repo.owntracks.org.gpg.key
echo "deb  http://repo.owntracks.org/debian wheezy main" > /etc/apt/sources.list.d/owntracks.list
apt-get update
apt-get install ot-recorder
```

### Installing on Debian 8 (Jessie)

```
wget http://repo.owntracks.org/repo.owntracks.org.gpg.key
apt-key add repo.owntracks.org.gpg.key
echo "deb  http://repo.owntracks.org/debian jessie main" > /etc/apt/sources.list.d/owntracks.list
apt-get update
apt-get install ot-recorder
```

#### sytemd service

The packages we provide have a systemd unit file in `/usr/share/doc/ot-recorder/ot-recorder.service` which you can use to have the Recorder started automatically:

1. Ensure you have a configuration file with the settings you require
2. `install -m444 /usr/share/doc/ot-recorder/ot-recorder.service /etc/systemd/system/ot-recorder.service`
3. Enable the service to run at startup: `systemctl enable ot-recorder`
4. Launch the service `systemctl start ot-recorder`


## Docker

We also have a Docker image to create containers which integrate a [Mosquitto broker](http://mosquitto.org) with the _Recorder_. The Docker image is [available from the Docker hub](https://hub.docker.com/r/owntracks/recorderd/) (e.g. `docker pull owntracks/recorderd`), and it's [usage is documented in the Booklet](http://owntracks.org/booklet/clients/recorder/).

## Tips and Tricks

### Gatewaying HTTP to MQTT

It actually is possible to gateway location publishes arriving via HTTP into MQTT, though you should be careful not to create loops. You can accomplish this with one or more Lua hooks using MQTT from within Lua.

### Override reverse-geo precision

If a payload is received with an element called `_geoprec` it contains an overide for the Recorder's configured reverse-geo precision. So, for example, if Recorder is running with precision 7, say, and the received payload contains `"_geoprec" : 2` the 2 will be used for this particular publish. This is not used in the OwnTracks apps, but it can be used with payloads you generate otherwise. If `_geoprec` is negative, new reverse geo lookups will not be performed, but cached entries of `abs(_geoprec)` will be used.

