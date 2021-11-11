..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 2

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Initial design for the EFD Transformation Service and stream processing tasks required to transform the raw EFD telemetry data into the format published in the Consolidated Database for the Science Platform users.


Introduction
============

The `Engineering and Facilities Database (EFD)`_ stores data primarily on `InfluxDB`_, an open source time series platform optimized for efficient storage and analysis of time series data.

At the Summit, InfluxDB + Chronograf + Kapacitor are used for real-time monitoring of the observatory systems. The raw data stream (Telemetry, Events, and Commands) is also `replicated to the LSST Data Facility (LDF)`_ using Kafka, and made available to the project at large through LDF EFD instance.

In this technote, we revisit the EFD transformation service, initially described in `DMTN-050`_.
The goal of the EFD transformation service is to curate the EFD telemetry data, relate it to the observations, and publish it into the Consolidated Database for the Science Platform users.

We review the EFD architecture and describe the components of the EFD transformation service. 

We suggest an approach for creating *transformed telemetry tables* in the Consolidated Database from *transformed telemetry streams* in Kafka which are produced by `kafka-aggregator`_ from Kafka *raw telemetry streams*.

Finally, we discuss extensions to the `kafka-aggregator`_ tool to produce the *transformed telemetry streams*.

.. _Engineering and Facilities Database (EFD): https://sqr-034.lsst.io
.. _InfluxDB: https://www.influxdata.com/time-series-database/
.. _replicated to the LSST Data Facility (LDF): https://sqr-050.lsst.io
.. _DMTN-050: https://dmtn-050.lsst.io
.. _kafka-aggregator: https://kafka-aggregator.lsst.io


Architecture
============

Figure 1 shows the components of the EFD transformation service.

.. figure:: /_static/efd_transformation_service.svg
   :name: EFD transformation service

   Components of the EFD transformation service



As of May 2021, `ts_xml`_ defines the schema for 248 Telemetry, 352 Commands, and 561 Event topics for 62 different observatory systems.

SAL translates the ``ts_xml`` schemas into IDL schemas used by DDS. The IDL schemas are then translated to Avro schemas and registered in the EFD by the SAL Kafka producers at the Summit.
The Avro schemas, Kafka topics configuration and the raw telemetry streams are replicated to the LDF EFD.
At LDF, we use Kafka Connect connectors to consume the Kafka stream, in particular, the raw telemetry stream is record in InfluxDB and in Parquet files.

The new components in the EFD architecture are i) `kafka-aggregator` which is responsible for consuming the *raw telemetry streams* and producing the *transformed telemetry streams*, and ii) the `JDBC Sink Connector` which is responsible to record the *transformed telemetry streams* into the Consolidated Database (PostgreSQL) for long-term persistence.

Once we have the  *transformed telemetry tables* in PostgreSQL, they can be accessed by the Science Platform users via the TAP service.

.. note::

   In this investigation, we assume that Science Platform users are interested only on telemetry data. We are not planning on recording commands and events in the Consolidated Database, however, see section :ref:`discussion`.

.. note::

   If we decide to record the *transformed telemetry streams* into InfluxDB or into Parquet files rather than in PostgreSQL, the `kafa-aggregator` work disscussed in section :ref:`kafka-aggregator-extensions` is still relevant.

Example: the WeatherStation data
================================

In this section, we show the schemas for the raw `WeatherStation` telemetry topics to illustrate the transformations required.

The ``WeatherStation`` subsystem has 12 telemetry topics:  ``airPressure``, ``airTemperature``,  ``dewPoint``, ``precipitation``, ``relativeHumidity``, ``snowDepth``, ``soilTemperature``, ``solarNetRadiation``, ``weather``, ``windDirection``, ``windGustDirection``, and ``windSpeed``. For simplicity, we only show the schemas for three of them:

.. csv-table:: WeatherStation.airTemperature
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "WeatherStationID","",""
   "avg1M","1 minute average value","deg_C"
   "avg24H","24 hours average","deg_C"
   "max24H","Maximum value during the last 24 hours","deg_C"
   "min24H","Minimum value during the last 24 hours","deg_C"
   "private_host","IP of sender","unitless"
   "private_identity","Identity of originator","unitless"
   "private_kafkaStamp","TAI time at which the Kafka message was created.","second"
   "private_origin","PID code of sender","unitless"
   "private_rcvStamp","TAI at receiver","second"
   "private_revCode","Revision code of topic","unitless"
   "private_seqNum","Sequence number","unitless"
   "private_sndStamp","TAI at sender","second"
   "private_efdStamp","UTC timestamp computed from private_sndStamp","second"
   "sensorName","Sensor model used to measure this parameters","unitless"

.. csv-table:: WeatherStation.dewPoint
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "WeatherStationID","",""
   "avg1M","1 minute average value","deg_C"
   "private_host","IP of sender","unitless"
   "private_identity","Identity of originator","unitless"
   "private_kafkaStamp","TAI time at which the Kafka message was created.","second"
   "private_origin","PID code of sender","unitless"
   "private_rcvStamp","TAI at receiver","second"
   "private_revCode","Revision code of topic","unitless"
   "private_seqNum","Sequence number","unitless"
   "private_sndStamp","TAI at sender","second"
   "private_efdStamp","UTC timestamp computed from private_sndStamp","second"
   "sensorName","Sensor model used to measure this parameters","unitless"

.. csv-table:: WeatherStation.windSpeed
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "WeatherStationID","",""
   "avg10M","10 minutes average value","m/s"
   "avg2M","2 minutes average value","m/s"
   "max10M","Maximum value during the last 10 minutes","m/s"
   "max2M","Maximum value during the last 2 minutes","m/s"
   "min2M","Minimum value during the last 2 minutes","m/s"
   "private_host","IP of sender","unitless"
   "private_identity","Identity of originator","unitless"
   "private_kafkaStamp","TAI time at which the Kafka message was created.","second"
   "private_origin","PID code of sender","unitless"
   "private_rcvStamp","TAI at receiver","second"
   "private_revCode","Revision code of topic","unitless"
   "private_seqNum","Sequence number","unitless"
   "private_sndStamp","TAI at sender","second"
   "private_efdStamp","UTC timestamp computed from private_sndStamp","second"
   "sensorName","Sensor model used to measure this parameters","unitless"
   "value","Instantaneous value","m/s"

A similar topic structure is seen accross all `Observatory subsystems`_.

If we simply reproduce the raw telemetry topics into the Consolidated Database we would have 248 individual tables, which makes it difficult to query and join with other tables in the Consolidated Database (e.g. the Exposure table).

.. _Observatory subsystems: https://ts-xml.lsst.io/sal_interfaces/index.html

Transformed telemetry tables
----------------------------

Let's use the ``WeatherStation`` data to examplify the creation of a *transformed telemetry table*.

.. csv-table:: Transformed WeatherStation telemetry table
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "timestamp", "Timestamp from the private_efdStamp field aggregated on 1 minute window."
   "airPressure.paAvg1M","1 minute average value for airPressure","hPa"
   "airTemperature.avg1M","1 minute average value for airTemperature","deg_C"
   "dewPoint.avg1M","1 minute average value for dewPoint","deg_C"
   "precipitation.prSum1M","1 minute sum value for precipitation","mm/h"
   "precipitation.prfSum1M","1 minute sum value for precipitation intensity","mm/h"
   "relativeHumidity.avg1M","1 minute average value for relativeHumidity","%"
   "snowDepth.avg1M","1 minute average value for snowDepth","cm"
   "soilTemperature.avg1M","1 minute average value for soilTemperature","deg_C"
   "solarNetRadiation.avg1M","1 minute average value for solarNetRadiation","W/m2"
   "weather.ambient_temp","The ambient temperature.","deg_C"
   "weather.humidity","The humidity.","%"
   "weather.pressure","The pressure outside.","hPa"
   "windDirection.value","1 minute average value for windDirection","deg"
   "windGustDirection.value10M","value for the last 10 minutes for windDirection","deg"
   "windSpeed.value","1 minute average value for windSpeed","m/s"

The rationale for the suggested schema is the following:

- The transformed ``WeatherStation`` telemetry table combine information from multiple raw ``WeatherStation`` telemetry topics.

- Fields that are not relevant to the Science Platform user can be excluded. In particular, most of the ``private_`` fields added by SAL can be excluded.

- In this example, the original topics have aggregated fields like ``min24H``, ``avg24H``, ``max24H``. We decided to keep only the "1 minute average" fields, which is the higher resolution available for all the ``WeatherStation`` telemetry topics.

Note: despite their names, the ``value`` and ``value10M`` fields for the ``windDirection``, ``windSpeed`` and ``windDirection`` topics also have 1 minute average values.

- In the transformed table, we decided to prefix the fields with the source ``WeatherStation`` topic name to identify its origin.

- The ``timestamp`` field in the transformed table is derived from the ``private_efdStamp`` field. The other timestamps are discarded.


Note: the timestamps from the raw ``WeatherStation`` telemetry topics are not necessarilly aligned, see section :ref:`joining` for details.

From this example, and also after looking at a handful of other T&S subsystems, we conclude that:

- the EFD transformation service must specify a mapping between the source telemetry topics and the *transformed telemetry table*, and which fields within those topics to use.

- in some cases the EFD transformation service needs to apply transformations to the fields' values, and must allow for new descriptions and units for the transformed fields.

The *EFD transformation service holds the decisions* necessary to create the transformed telemetry tables from the raw telemetry topics.

Advantages
----------

Some advantages of this approach:

- Instead of 249 tables we might have 62, one per subsystem, or even less. By reducing the number of tables in the Consolidated Database we simplify the queries considerably and reduce the number of joins and relations in the database.

- By exposing only the relevant information to the Science Platform user, we also reduce the amount of data in the Consolidated Database making it more managable over time.

- By transforming field values, we add value and make the EFD telemetry data easier to use.

- Another task of the EFD transformation service is the aggregation of field values over time for high-frequency telemetry streams, which also reduces the amount of data in the Consolidated Database to a great extent.

In the following sections, we describe the Kafka Connect JDBC Sink connector and ``kafka-aggregator``. We try to use the JDBC Sink connector features as much as possible, and delegate to ``kafka-aggregator`` the functionalities that cannot be performed by the connector.

.. _ts_xml: https://ts-xml.lsst.io/sal_interfaces

The Kafka Connect JDBC Sink connector
=====================================

In this section, we describe some features of the `Kafka Connect JDBC Sink connector`_ and how it is used in the EFD transformation service.

.. _Kafka Connect JDBC Sink connector: https://docs.confluent.io/kafka-connect-jdbc/current/sink-connector/index.html

Schema creation
---------------

The  `Kafka Connect JDBC Sink connector`_ *requires an explicit schema* to automatically create a table in a relational database.
In the EFD, we accomplish that by using Avro and storing the Avro schemas in the Confluent Schema Registry.

Data types
^^^^^^^^^^

The JDBC Sink connector is responsible for `translating Avro data types to PostgresSQL data types`_, and it provides mechanisms to change data types explicilty for certain fields before creating the table schema (e.g. the ``timestamp`` field).

.. _translating Avro data types to PostgresSQL data types: https://docs.confluent.io/5.4.2/connect/kafka-connect-jdbc/sink-connector/index.html#auto-creation-and-auto-evoluton

Schema evolution
----------------

The JDBC Sink connector has limited support to `schema evolution`_, but it supports *forward compatible* schema changes with PostreSQL and can automatically issue an ``ALTER TABLE`` to add new columns to an existing table.

.. _schema evolution: https://docs.confluent.io/5.4.2/connect/kafka-connect-jdbc/sink-connector/sink_config_options.html#ddl-support

JDBC Sink transforms
--------------------

Flattening nested fields
^^^^^^^^^^^^^^^^^^^^^^^^

Support to ``ARRAY`` data type in PostgresSQL was `added just recently`_ to the JDBC Sink Connector, and may still have issues. Another approach is to use the ``flatten`` JDBC Sink transform to take a nested structure like an array and "flatten" it out.

.. code-block:: json

   'transforms'                          = 'flatten',
   'transforms.flatten.type'             = 'org.apache.kafka.connect.transforms.Flatten$Value'


.. _added just recently: https://github.com/confluentinc/kafka-connect-jdbc/pull/805

Handling timestamps
^^^^^^^^^^^^^^^^^^^

In ``ts_xml``, timestamps are Unix timestamps in secods with ``double`` (64-bit) precision. In the Consolidated Database, we want timestamps created with a proper data type to use SQL functions to operate with timestamps.
The ``setTimestampType`` JDBC Sink transform can be used to change the data type for the ``timestamp`` field in the *transformed telemetry tables*.

.. code-block:: json

   'transforms.setTimestampType.type'        = 'org.apache.kafka.connect.transforms.TimestampConverter$Value',
   'transforms.setTimestampType.field'       = 'timestamp',
   'transforms.setTimestampType.target.type' = 'Timestamp'

Declaring primary keys
----------------------

The natural choice for the primary key in the `transformed telemetry tables` is the ``timestamp`` field.
To do that, ``pk.mode`` must be set to ``record_value`` to use one or more fields as primary key.

.. code-block:: json

   'pk.mode'                                 = 'record_value',
   'pk.fields'                               = 'timestamp',


Working with multiple tables
----------------------------

When `working with multiple tables`_, database ingestion can be improved by adding more Kafka Connect workers.
There are two ways of doing this within the Kafka Connect framework.
One is to define multiple connectors, one for each table.
The other is to create a single connector but increase the number of connector tasks.

With the InfluxDB Sink, MirrorMaker 2 and S3 Sink connectors, creating a single connector and increasing the number of connector tasks seems enough to handle the current data throughput in the EFD.
This should work with the JDBC Sink connector too, as long as we can use the same connector configuration for all tables.

.. _working with multiple tables: https://www.confluent.io/blog/kafka-connect-deep-dive-jdbc-source-connector/#multiple-tables


.. _kafka-aggregator-extensions:

Transformed telemetry streams
=============================

A table is the materialization of a stream. In the previous section, we showed how the JDBC Sink connector can be used to create the *transformed telemetry tables* in a relational database like PostgreSQL.

In this section, we discuss extensions to `kafka-aggregator`_ to produce the *transformed telemetry streams*.

Kafka-aggregator
----------------

`kafka-aggregator`_ is based on `Faust`_, a Python Stream Processing library.
It implements Faust agents that subscribe to one kafka topic and produce a new aggregated topic.

In the current implementation `kafka-aggregator`_ computes summary statistics on `tumbling windows`_ (window aggregation), where the size of the window sets the frequency of the aggregated stream.
However, it can aggregate only one source topic and produced one aggregrated topic at the time (the 1:1 case).

The main extensions to `kafka-aggregator`_ include the ability of joining multiple Kafka topics into one source stream, and transformations like filtering and mapping in addition to window aggregation.
All of these transformations are chained together to produce the `transformed teletemery streams`.

.. note::

   Despite of these changes in `kafka-aggregator`_ we decided to keep its name because combining information from multiple source topics into a single stream is also a form of aggregation.

.. _tumbling windows: https://faust.readthedocs.io/en/latest/userguide/tables.html#windowing


.. _joining:

Joining
-------

With `Faust`_, it is possible to subscribe to multiple source topics by listing them in the `topic description`_.
Faust also supports other `join strategies`_.

.. note::

   Expand this section after doing a proof of concept using Faust to join the source streams.

.. _topic description: https://faust.readthedocs.io/en/latest/userguide/agents.html#the-channel
.. _join strategies: https://faust.readthedocs.io/en/latest/reference/faust.joins.html?highlight=join


Filtering and mapping
---------------------

The extended `kafka-aggregator`_ requires a new mechanism to define the schema for the aggregated topics.
In the new implementation, `kafka-aggregator`_ specify the source Kafka topics to an aggregated Kafka topic, by explicilty listing the source topics and fields to use.

In the same mapping configuration, we can specify functions to transform the field values (map), if needed, and enable or disable window aggregation on fields.

We propose replacing some the `kafka-aggregator settings`_ by a configuration file like the following:

.. code-block:: yaml

   ---
   aggregated_topic_name1:
      filter:
         source_topic_name1:
            field1:
               name: new_name
               description: "new description for the transformed field"
               units: "new units for the transformed field"
               map: func1
            field2:
               description: "new description for the transformed field"
               units: "new units for the transformed field"
               map: func2
            field3:
            ...
         source_topic_name2:
            field1:
            field2:
            field3:
            ...
         ...
   aggregated_topic_name2:
      window_aggregation_size: 1s
      window_expiration_seconds: 1s
      operations:
         - min
         - median
         - max
      filter:
         source_topic_name3:
            field1:
            field2:
            field3:
            ...
         ...
   ...

In this configuration, we specify the aggregated topic (the new topic in Kafka where the aggregated data is produced to), the source topics in Kafka to consume from, and the fields within those topics to use.

For each field, we can specify optionally a name, a description, units and a function (map) to transform its values.
If not specified, the default field name, description, and units are obtained from the source topic schema.
If a map is not specified, the value from the source field is used.

Window aggregation
------------------

For each aggregated topic we can optionally specify the window aggregation transformation, configuring the tumbling window parameters and the summary statistics to compute (operations).

The allowed operations are ``min``, ``q1``, ``mean``, ``median``, ``q3``, ``stdev`` and ``max``.

The window aggregation is computed after the map transformation if any.

.. _kafka-aggregator settings: https://kafka-aggregator.lsst.io/v/dependabot-docker-python-3.9.5-buster/configuration.html#kafka-aggregator-settings
.. _aggregated topic name: https://kafka-aggregator.lsst.io/configuration.html#aggregation-topic-name
.. _excluded field names: https://kafka-aggregator.lsst.io/configuration.html#special-field-names
.. _operations: https://kafka-aggregator.lsst.io/configuration.html#summary-statistics
.. _Aggregator class: https://kafka-aggregator.lsst.io/api/kafkaaggregator.aggregator.Aggregator.html#kafkaaggregator.aggregator.Aggregator
.. _Faust: https://faust.readthedocs.io/en/latest/

Relating Telemetry data with the observations
=============================================

.. note::

   It is not clear how the Exposure table is created in the Consolidated Database (see section :ref:`discussion`).
   Assuming it exists, we need an additional step to create a constraint on the *transformed telemetry tables* that references the Exposure table primary key, or intermediate tables to hold the relationship between the *transformed telemetry tables* and the Exposure table.
   Need to expand this section further.

.. _discussion:

Discussion
==========

**Why publishing only EFD telemetry data to the Consolidated Database?**

The EFD data comprises telemetry, Events, and Commands topics.
While Events and Commands are crucial for engineers in understanding the telescope systems during operations, they are less critical to science users.
Telemetry is essential for science users to correlate with data quality after data acquisition and data processing.

**What happens if the science user needs data from the EFD that is not published to the Consolidated Database?**

That is a common problem of defining a schema upfront and perhaps the most sensitive aspect of EFD transformation service design.

The proposed solution is flexible enough to allow changes to the EFD Consolidated Database schema that are *forward compatible*. It is possible to add new tables and new columns to existing tables in the Consolidated Database at any given time.
The forward compatibility of the EFD Consolidated Database schema ensures that queries that worked with the old schema continue to work with the new schema.
Similarly, queries designed to work with the new schema only return meaningful values for data inserted *after* the schema change.

The above may represent a limitation for the current solution because the proposed process will not perform a batch load of the historical EFD data when the Consolidated Database schema changes.
Replay the raw EFD data from Parquet files to Kafka might be an option, but it is out of the scope of this implementation.

**Is the EFD transformation service also responsible for creating "Exposure tables" for the Auxiliary Telescope and the Main Telescope in the Consolidated Database?**

DMTN-050 mentions relations between the telemetry tables and Exposure tables, but it is not clear who is responsible for creating the latter.

In principle, the ``ATExposure`` and ``MTExposure`` tables in the Consolidated Database can be derived from the ``ATCamera_logevent_endReadout`` and ``MTCamera_logevent_endReadout`` Events. When these events are received, the corresponding images should be complete.

.. csv-table:: ATCamera.logevent_endReadout
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "additionalKeys","Additional keys passed to the takeImages command (: delimited)","unitless"
   "additionalValues","Additional values passed to the takeImages command (: delimited; in same order as additionalKeys)","unitless"
   "imageController","The controller for the image  (O=OCS/C=CCS/...)","unitless"
   "imageDate","The date component of the image name (YYYYMMDD)","unitless"
   "imageIndex","The zero based index number for this specific exposure within the sequence","unitless"
   "imageName","The imageName for this specific exposure; assigned by the camera","unitless"
   "imageNumber","The image number (SEQNO) component of the image name","unitless"
   "imageSource","The source component of the image name (AT/CC/MC)","unitless"
   "imagesInSequence","The total number of requested images in sequence","unitless"
   "priority","Priority code","unitless"
   "private_host","IP of sender","unitless"
   "private_identity","Identity of originator","unitless"
   "private_kafkaStamp","TAI time at which the Kafka message was created.","second"
   "private_origin","PID code of sender","unitless"
   "private_rcvStamp","TAI at receiver","second"
   "private_revCode","Revision code of topic","unitless"
   "private_seqNum","Sequence number","unitless"
   "private_sndStamp","TAI at sender","second"
   "requestedExposureTime","The requested exposure time (as specified in the takeImages command)","second"
   "timestampAcquisitionStart","The effective time at which the image acquisition started (i.e. the end of the previous clear or readout)","second"
   "timestampEndOfReadout","The time at which the readout was completed","second"

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
