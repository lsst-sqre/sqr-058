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

   Initial design for the EFD schema in the consolidated database, and stream processing tasks required to transform the raw telemetry data into the version published to the Science Platform users.


Introduction
============

The current implementation of the `Engineering and Facilities Database (EFD)`_ is based on InfluxDB, an open source time series platform optimized for efficient storage and analysis of time series data.

While the Summit EFD instance is used for real-time monitoring of the Observatory, the raw data stream (Telemetry, Events, and Commands) is also `replicated to the LSST Data Facility (LDF)`_, and accessible to the project at large via the LDF EFD instance.

In this technote, we revisit the EFD transformation service, initially described in `DMTN-050`_.
The main goal of the EFD transformation service is to curate the EFD telemetry data, relate it to the observations, and publish it to the Consolidated Database enabling access to the Science Platform science users.

In particular, we review the EFD architecture and the components of the EFD transformation service. 
We analyze the schemas of telemetry topics from different Observatory subsystems and suggest an approach for creating *transformed telemetry tables* in the Consolidated Database.

We show how the Kafka Connect JDBC Sink connector can create, evolve the schema, and write to the *transformed telemetry tables* consuming the *transformed telemetry streams*.

Finally, we discuss how to extend the `kafka-aggregator`_ tool to produce the *transformed telemetry streams*.

.. _Engineering and Facilities Database (EFD): https://sqr-034.lsst.io
.. _replicated to the LSST Data Facility (LDF): https://sqr-050.lsst.io
.. _DMTN-050: https://dmtn-050.lsst.io
.. _kafka-aggregator: https://kafka-aggregator.lsst.io


Architecture
============

Figure 1 shows the components of the EFD transformation service at LDF.

.. figure:: /_static/efd_transformation_service.svg
   :name: EFD transformation service

   Components of the EFD transformation service


EFD telemetry topics
====================

May 10, 2021, `ts_xml`_ defines the schema for 249 Telemetry, 390 Commands, and 533 Events topics for 62 different Observatory subsystems.

SAL translates the ``ts_xml`` schemas for each topic into IDL schemas used by DDS. The IDL schemas are then translated to Avro schemas and registered in the EFD when the SAL Kafka producers are initialized.

In this investigation, we assume that the leading interest of Science Platform science users is the EFD Telemetry data, and that *Events and Commands do not need to be recorded into the Consolidated Database* (see section :ref:`discussion`).

In this section, we analyze the schemas for several `Observatory subsystems`_ and use the `WeatherStation` subsystem as an example.

Example: the WeatherStation subsystem
-------------------------------------

The ``WeatherStation`` subsystem has 12 telemetry topics:  ``airPressure``, ``airTemperature``,  ``dewPoint``, ``precipitation``, ``relativeHumidity``, ``snowDepth``, ``soilTemperature``, ``solarNetRadiation``, ``weather``, ``windDirection``, ``windGustDirection``, and ``windSpeed``. For simplicity, we show the schemas for 3 of them:

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
   "sensorName","Sensor model used to measure this parameters","unitless"
   "value","Instantaneous value","m/s"

A similar topic structure is seen in all the `Observatory subsystems`_.
If we simply reproduce the raw EFD telemetry topics into the Consolidated Database we would have 249 individual tables that would be hard to query.

The EFD transformation service is an opportunity to curate the raw EFD telemetry data and publish it to the Science Platform science users in a more meaningful manner.

In the next section we discuss our approach for creating the *Transformed telemetry tables* in the Consolidated Database.

.. _Observatory subsystems: https://ts-xml.lsst.io/sal_interfaces/index.html

Transformed telemetry tables
============================

Let's use the ``WeatherStation`` telemetry topics to examplify the creation of a *transformed telemetry table*.

.. csv-table:: Transformed WeatherStation telemetry table
   :header: "Name", "Description", "Units"
   :widths: 15, 30, 5

   "timestamp", "Average timestamp from private_sndStamp in UTC"
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
   "windDirection.avg2M","2 minutes average value for windDirection","deg"
   "windGustDirection.value10M","value for the last 10 minutes for windDirection","deg"
   "windSpeed.avg2M","2 minutes average value for windSpeed","m/s"


- The transformed ``WeatherStation`` telemetry table combines information from multiple ``WeatherStation`` telemetry topics.

- Fields that are not relevant to the Science Platform science user are excluded. In particular, most of the ``private_`` fields added by SAL can be excluded and others reduced to a *single* ``timestamp`` field.

- In this particular example, the original topics have aggregated fields like ``min24H``, ``avg24H``, ``max24H``. We decided to keep only the fields with "1 minute average values", which are available in most of the cases, and leave it up to the user to compute aggregations in SQL as needed.

- Field names are namespaced to identify the original EFD topic.

From this example, we conclude that to create a *transformed telemetry table*, the EFD transformation service must be able to specify a mapping between the source telemetry topics and the *transformed telemetry table*, and specify which fields within those topics to use.
In some cases, it must be able to apply transformations to the fields' values, and allows for new descriptions and units for the transformed fields.

In other words, the EFD transformation service holds the transformations (decisions) necessary to create the Consolidated Database telemetry tables from the raw EFD telemetry topics.

Advantages
----------

Some advantages of this approach:

- Instead of 249 tables we might have 62, one per subsystem, or even less. By reducing the number of tables in the Consolidated Database we simplify the queries considerably by avoiding multiple joins. It also simplifies creating relations in the database among the telemetry tables and the Exposure table.

- By exposing only the relevant information to the Science Platform science user, we also reduce the amount of data in the Consolidated Database making it more managable over time.

- By transforming field values, we add value and make the EFD telemetry data easier to use.

- Another task of the EFD transformation service is the aggregation of field values over time for high-frequency telemetry streams, which also reduces the amount of data in the Consolidated Database to a great extent.

In the following sections, we describe the Kafka Connect JDBC Sink connector and the ``kafka-aggregator`` tool. We try to use the JDBC Sink connector functionalities as much as possible, and delegate to the ``kafka-aggregator`` tool the functionalities that cannot be performed by the connector.

.. _ts_xml: https://ts-xml.lsst.io/sal_interfaces
.. _planned to be in UTC: https://jira.lsstcorp.org/browse/RFC-767

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

In ``ts_xml``, timestamps are Unix timestamps with millisecond precision and have ``double`` (64-bit) types. In the Consolidated Database, we want timestamps created with a proper data type to use SQL functions to operate with timestamps.
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

When `working with multiple tables`_, the ingestion time in the Consolidated Database can be reduced by addind more Kafka Connect workers.
There are two ways to do this with the Kafka Connect framework.
One is to define multiple connectors, one for each table.
The other is to create a single connector but increase the number of connector tasks.

With the InfluxDB Sink and MirrorMaker 2 connectors, creating a single connector and increasing the number of connector tasks works fine to handle the current data throughput in the EFD.
This should work with the JDBC Sink connector too, as long as we can use the same connector configuration with all the *transformed telemetry tables*.

.. _working with multiple tables: https://www.confluent.io/blog/kafka-connect-deep-dive-jdbc-source-connector/#multiple-tables

Transformed telemetry streams
=============================

A table is the materialization of a stream. In the previous section, we showed how the JDBC Sink connector can be used to create the *transformed telemetry tables*.

In this section, we discuss how to extend the `kafka-aggregator`_ tool to produce the *transformed telemetry streams*.

Kafka-aggregator
----------------

The `kafka-aggregator`_ tool is based on `Faust`_, a Python Stream Processing library.
It implements Faust agents that consume a source topics from Kafka and produce a new aggregated topics.

The aggregated topic schema is created based on the source topic schema with some support to `exclude fields`_.
The result is a new aggregated stream with new aggregated fields where the size of the aggregation window sets the frequency of the stream.

.. _exclude fields: https://kafka-aggregator.lsst.io/configuration.html#kafka-aggregator-settings

In the EFD transformation service, this can be optional, e.g., low frequency streams like the transformed ``WeatherStation`` telemetry stream do not need further aggregation.

The above suggests that `kafka-aggregator` could be extended to produce the *transformed telemetry topic* and that computing window aggregations should be an optional step.

.. note::

   We decided to keep the name `kafka-aggregator`_ for the extended tool because joining related streams to produce a single stream is also a form of aggregation.


Joining source streams
----------------------

With `Faust`_, it is possible to subscribe to multiple source topics by listing them in the `topic description`_.
Faust also supports different `join strategies`_.

.. note::

   Expand this section after doing a proof of concept using Faust to join the source streams.

.. _topic description: https://faust.readthedocs.io/en/latest/userguide/agents.html#the-channel
.. _join strategies: https://faust.readthedocs.io/en/latest/reference/faust.joins.html?highlight=join


Mapping and transformation
--------------------------

`kafka-aggregator`_ requires a new mechanism to define the schema for the aggregated topics.
In this implementation, `kafka-aggregator`_ configures a mapping of source topics to an aggregated topic.
In particular, this implementation replaces the configuration options to exclude topics and fields from aggregation, and explicitly lists the source topics and the fields used in the mapping instead.

In the same mapping configuration, we can specify functions to transform the field values, if needed, and enable or disable window aggregation on fields.

We propose replacing the `kafka-aggregator settings`_ by an YAML file like the following:

.. code-block:: yaml

   ---
   aggregated_topic_name1:
      mapping:
         source_topic_name1:
            field1:
               name: new_name
               description: "new description for the transformed field"
               units: "new units for the transformed field"
               transformation: func1
            field2:
               description: "new description for the transformed field"
               units: "new units for the transformed field"
               transformation: func2
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
      operations:
         - min
         - median
         - max
      mapping:
         source_topic_name3:
            field1:
            field2:
            field3:
            ...
         ...
   ...

In this YAML file, we specify the aggregated topics (the destination topics in Kafka where the *transformed telemetry streams* are produced to), the source topics in Kafka to consume from, and the fields within those topics to use.

For each field in the aggregated topic, we can specify optionally a name, adescription, units and a transformation function.
If not specified, the default field name, description and units are obtained from the source topic schema.
If a transformation function is specified, it is used to transform the field values.

The ``window_aggregation_size`` configuration can be specified in the YAML file per aggregated topic, indicating that the summary statistics operations configured in ``operations`` should be computed for each numeric field in the mapping after the transformation is applied, if any.
Currently, the allowed summary statistics computed by ``kafka-aagregator`` are ``min``, ``q1``, ``mean``, ``median``, ``q3``, ``stdev`` and ``max``.

Finally, we expect to reuse the `Aggregator class`_ in `kafka-aggregator`_ to create the Faust-avro record and the Avro schema for the aggregated topic with little modification.

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

That is a common problem of designing a schema upfront and perhaps the most sensitive aspect of EFD transformation service.

The proposed solution is flexible enough to allow changes to the EFD Consolidated Database schema that are *forward compatible*. It is possible to add new tables and columns to existing tables in the Consolidated Database at any given time.
The forward compatibility of the EFD Consolidated Database schema ensures that queries that worked with the old schema continue to work with the new schema.
Similarly, queries designed to work with the new schema only return meaningful values for data inserted *after* the schema change.

The above may represent a limitation for the current solution because the proposed process will not perform a batch load of the historical EFD data when the Consolidated Database schema changes.
Replay the raw EFD data from Parquet files to Kafka might be an option, but it is out of the scope of this implementation.

**Is the EFD transformation service also responsible for creating "Exposure tables" for the AT and the MT in the Consolidated Database?**

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
