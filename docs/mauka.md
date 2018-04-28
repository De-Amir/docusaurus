---
title: OPQ Mauka
sidebar_label: Mauka
---

OPQ Mauka is a distributed plugin-based middleware for OPQ that provides higher level analytic capabilities. OPQ Mauka provides analytics for classification of PQ events, aggregation of triggering data for long term trend analysis, community detection, statistics, and metadata management. It is intended to provide the following capabilities:

* Recording long term trends from triggering measurements
* Classification of voltage dip/swells
* Classification of frequency dip/swells
* Requests of raw data for higher level analytics, including:
  * Community detection
  * Grid topology
  * Global/local event detection/discrimination
  * Integration with other data sources (_i.e._ PV production) 

## Installation

### Install Python

OPQ Mauka requires version **3.5 or greater** of Python. It's suggested that you use your distribution's package manager to install Python 3.5 or greater if its available. Python 3.5 or greater can also be downloaded [here](https://www.python.org/).

### Install Python dependencies

1. Use pip to automatically install the dependencies for this project by referencing the ```mauka/requirements.txt``` file
```
pip install -r requirements.txt
```

### Run Mauka as service (Debian based systems)

If you would like Mauka to start at boot, you must create a service for it. This documentation assumes that SysVinit is used and the start-stop-daemon binary is available (most modern Debian based distributions).

1. Run the script ```util/mauka/mauka-install.sh``` as root
2. The service will now start automatically at boot
3. To start the service type ```sudo service mauka start```
4. To stop the service type ```sudo service mauka stop```
5. To restart the service type ```suer service mauka restart```


*Note: If you successfully create services for OS X or systemd, please let us know!*

### Run Mauka manually (non-Debian based systems)

1. Run ```mauka/OpqMauka.py```
```
# Enter the opq/mauka directory
cd mauka

# Run Mauka
python3 OpqMauka.py path_to_config.json

```

## Design

OPQMauka is written in Python 3 and depends on 2 ZMQ brokers as well as a Mongo database. The architecture of OPQMauka is designed in such a way that all analytics are provided by plugins that communicate using publish/subscribe semantics using ZMQ. This allows for a distributed architecture and horizontal scalability. 

The OPQMauka processing pipeline is implemented as a directed acyclic graph (DAG). Communication between vertexes in the graph is provided via ZeroMQ. Each node in the graph is implemented by an OPQMauka Plugin. Additional analysis plugins can be added to OPQMauka at runtime, without service interruption.

Each OPQMauka Plugin provides a set of topics that it subscribes to and a set of topics that it produces. These topics form the edges between vertexes in our graph. Because each plugin is independent and only relies on retrieving and transmitting data over ZeroMQ, plugins can be implemented in any programming language and executed on any machine in a network. This design allows us to easily scale plugins across multiple machines in order to increase throughput.

The following image illustrates the relationship between Mauka and its plugins:

<img src="/docs/assets/mauka/opqmauka.png" width="400px">

## Plugins

### Base Plugin

The [base plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/base.py) is a base class which implements common functionally across all plugins. This plugin in subclassed by all other OPQMauka plugins. The functionality this plugin provides includes:

* Access to the underlying Mongo database
* Automatic publish subscribe semantics with ```on_message``` and ```publish``` APIs (via ZMQ)
* Configuration/JSON parsing and loading
* Python multiprocessing primitives 
* Status/heartbeat notifications

### Measurement Plugin

The [measurement plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/MeasurementPlugin.py) records raw triggering data and aggregates it into a measurements collection in our Mongo database. 

These measurements are mainly used for analyzing long term trends and for display in OPQView. It's possible to control the sampling of raw triggering messages by setting the ```plugins.MeasurementPlugin.sample_every``` key in the configuration file.

### Threshold Plugin

The [threshold plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/ThresholdPlugin.py) is a base plugin that implements functionality for determining preset threshold crossings over time. That is, this plugin, given a steady state, will detect deviations from the steady using percent deviation from the steady state as a discriminating factor. 

When subclassing this plugin, other plugins will define the steady state value, the low threshold value, the high threshold value, and the duration threshold value.

This plugin is subclassed by the voltage and frequency threshold plugins.

Internally, the threshold plugin looks at individual measurements and determines if the value is low, stable, or high (as defined by the subclass). A finite state machine is used to switch between the following states and define events.

**Low to low.** Still in a low threshold event. Continue recording low threshold event.

**Low to stable.** Low threshold event just ended. Produce an event message.

**Low to high.** Low threshold event just ended. Produce and event message. Start recording high threshold event.

**Stable to low.** Start recording low threshold event.

** Stable to stable.** Steady state. Nothing to record.

**Stable to high.** Start recording high threshold event.

**High to low.** High threshold event just ended. Produce event message. Start recording low threshold event.

**High to stable.** High threshold event just ended. Produce event message.

**High to high.** Still in high threshold event. Continue recording event. 

Event messages are produced by passing the contents of a recorded event to the ```on_event``` method call. This method call needs to be implemented in all subclassing plugins in order to deal with the recorded event.

### Frequency Threshold Plugin

The [frequency threshold plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/FrequencyThresholdPlugin.py) subclasses the threshold plugin and classifies frequency dips and swells.

By default, this plugin assumes a steady state of 60Hz and will detect dips and swells over 0.5% in either direction. The thresholds can be configured by setting the keys ```plugins.ThresholdPlugin.frequency.ref```, ```plugins.ThresholdPlugin.frequency.threshold.percent.low```, and ```plugins.ThresholdPlugin.frequency.threshold.percent.high``` in the configuration file.

When thresholds are tripped, frequency events are generated and published to the system. These are most importantly used to generate event triggering requests to OPQMauka to request raw data from affected devices.

### Voltage Threshold Plugin

The [voltage threshold plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/VoltageThresholdPlugin.py) subclasses the threshold plugin and classifies voltage dips and swells.

By default, this plugin assumes a steady state of 120hz and will detect dips and swells over 5% in either direction. The thresholds can be configured by setting the keys plugins.ThresholdPlugin.voltage.ref, plugins.ThresholdPlugin.voltage.threshold.percent.low, and plugins.ThresholdPlugin.voltage.threshold.percent.high in the configuration file.

When thresholds are tripped, voltage events are generated and published to the system. These are most importantly used to generate event triggering requests to OPQMauka to request raw data from affected devices.

### Total Harmonic Distortion Plugin
The [total harmonic distortion (THD) plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/ThdPlugin.py) subscribes to all events that request data, waits until the data is realized, performs THD calculations over the data, and then stores the results back to the database.

This plugin subscribes to events that request data and also THD specific messages so that this plugin can be triggered to run over historic data as well. The amount of time from when this plugin receives a message until it assumes the data is in the database can be configured in the configuration file. 

The THD calculations are computed in a separate thread and the results are stored back to the database. 

### ITIC Plugin 

The [ITIC plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/IticPlugin.py) subscribes to all events that request data, waits until the data is realized, performs ITIC calculations over the data, and then stores the results back to the database.

This plugin subscribes to events that request data and also ITIC specific messages so that this plugin can be triggered to run over historic data as well. The amount of time from when this plugin receives a message until it assumes the data is in the database can be configured in the configuration file. 

The ITIC calculations are computed in a separate thread and the results are stored back to the database. 

ITIC regions are determined by plotting the curve and performing a point in polygon algorithm to determine which curve the point falls within.


### Acquisition Trigger Plugin

The [acquistion trigger plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/AcquisitionTriggerPlugin.py) subscribes to all events and forms event request messages to send to OPQMakai to enable the retrieval of raw power data for higher level analytics.

This plugin employs a deadzone between event messages to ensure that multiple requests for the same data are not sent in large bursts, overwhelming OPQBoxes or OPQMakai. The deadzone by default is set to 60 seconds, but can be configured by setting the ```plugins.AcquisitionTriggerPlugin.sDeadZoneAfterTrigger``` key in the configuration. If this plugin encounters an event while in a deadzone, a request is still generated and sent to OPQMakai, however a flag is set indicating to Makai that raw data should not be requested.


### Status Plugin

The [status plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/StatusPlugin.py) subscribes to heatbeat messages and logs heartbeats from all other plugins (including itself).

### Print Plugin

The [print plugin](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/PrintPlugin.py) subscribes to all topics and prints every message. This plugin is generally disabled and mainly only useful for debuggin purposes.

## Plugin Development

The following steps are required to create a new OPQMauka plugin:

1. Create a new Python module for the plugin in the plugins package (i.e. MyFancyPlugin.py).

2. import the plugin base
```
import plugins.base
```

3. Create a class that extends the base plugin.
```
class MyFancyPlugin(plugins.base.MaukaPlugin):
      ...
```

4. Create the following module level function
```
def run_plugin(config):
      plugins.base.run_plugin(MyFancyPlugin, config)
```

5. Provide the following constructor for your class. Ensure the a call to super provides the configuration, list of topics to subscribe to, and the name of the plugin.
```
def __init__(self, config):
      super().__init__(config, ["foo", "bar"], "MyFancyPlugin")
```

6. Overload the ```on_message``` from the base class. This is how you will receive all the messages from topics you subscribe to.
```
def on_message(self, topic, message):
      ...
```

7. Produce messages by invoking the superclasses produce method.
```
self.produce("topic", "message")
```

8. Import and add your plugin in plugins/```__init__.py```.
```
from plugins import MyFancyPlugin
```

9. Add your plugin to the plugin list in ```OpqMauka.py```.

An example plugin template might look something like:

```
# plugins/MyFancyPlugin.py
import plugins.base

def run_plugin(config):
    plugins.base.run_plugin(MyFancyPlugin, config)

def MyFancyPlugin(plugins.base.MaukaPlugin):
    def __init__(self, config):
         super().__init__(config, ["foo", "bar"], "MyFancyPlugin")

    def on_message(self, topic, message):
          print(topic, message)
```

## Message Injection

It's nice to think of Mauka as a perfect DAG of plugins, but sometimes its convenient to dynamically publish (topic, message) pairs directly into the Mauka system.

This is useful for testing, but also useful for times when we want to run a plugin of historical data. For instance, let's say a new plugin Foo is developed. In order to apply Foo's metric to data that has already been analyzed, we can inject messages into the system targetting the Foo plugin and triggering it to run over the old data.

This functionality is currently contained in [plugins/mock.py](https://github.com/openpowerquality/opq/blob/master/mauka/plugins/mock.py).

The script can either be used as part of an API or as a standalone script. As long as the URL of Mauka's broker is known, we can use this inject messages into the system. This provides control over the topic, message contents, and the type of the contents (string or bytes). 