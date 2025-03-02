.. role:: javascript(code)
  :language: javascript

==================
Command Monitoring
==================

:Title: Command Monitoring
:Status: Approved
:Minimum Server Version: 2.4

.. contents::

--------

Specification
=============

The performance monitoring specification defines a set of behaviour in the drivers for providing runtime information about commands to any 3rd party APM library as well internal driver use, such as logging.

-----------
Definitions
-----------

META
----

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.


Terms
-----

Document
  The term ``Document`` refers to the implementation in the driver's language of a BSON document.

--------
Guidance
--------

Documentation
-------------

The documentation provided in code below is merely for driver authors and SHOULD NOT be taken as required documentation for the driver.


Operations
----------

All drivers MUST implement the API. Implementation details are noted in the comments when a specific implementation is required. Within each API, all methods are REQUIRED unless noted otherwise in the comments.


Operation Parameters
--------------------

All drivers MUST include the specified parameters in each operation, with the exception of the options parameter which is OPTIONAL.


Naming
------

All drivers MUST name operations, parameters and topic names as defined in the following sections. Exceptions to this rule are noted in the appropriate section. Class and interface names may vary according to the driver and language best practices.


Publishing & Subscribing
------------------------

The driver SHOULD publish events in a manner that is standard to the driver's language publish/subscribe patterns and is not strictly mandated in this specification.


Guarantees
----------

The driver MUST guarantee that every ``CommandStartedEvent`` has either a correlating ``CommandSucceededEvent`` or ``CommandFailedEvent``.

The driver MUST guarantee that the ``requestId`` of the ``CommandStartedEvent`` and the corresponding ``CommandSucceededEvent`` or ``CommandFailedEvent`` is the same.

Unacknowledged/Acknowledged Writes
----------------------------------

 Unacknowledged writes must provide a ``CommandSucceededEvent`` with a ``{ ok: 1 }`` reply.

A non-default write concern MUST be included in the published command. The default write concern is not required to be included.

Succeeded or Failed
-------------------

Commands that executed on the server and return a status of ``{ ok: 1.0 }`` are considered
successful commands and MUST fire a ``CommandSucceededEvent``. Commands that have write errors
are included since the actual command did succeed, only writes failed.

``CommandSucceededEvent`` and ``CommandFailedEvent`` events MUST have a ``requestId`` that matches their
originating ``CommandStartedEvent``.

Error Handling
--------------

If an exception occurs while sending the operation to the server, the driver MUST generate a ``CommandFailedEvent`` with the exception or message and re-raise the exception.

Bulk Writes
-----------

This specification defines the monitoring of individual commands and in that respect MUST generate
an event for each command a bulk write executes. Each of these commands, however, must be linked
together via the same ``operationId``.

Implementation Notes
--------------------

When a driver sends an OP_MSG with a document sequence, it MUST include the document sequence as a BSON array in CommandStartedEvent.command. The array's field name MUST be the OP_MSG sequence identifier. For example, if the driver sends an "update" command using OP_MSG, and sends a document sequence as a separate section of payload type 1 with identifier "updates", the driver MUST include the documents as a BSON array in CommandStartedEvent.command with field name "updates".

See "Why are document sequences included as BSON arrays?" in the `rationale`_.

---------
Rationale
---------

*1. Why are commands with* ``{ ok: 1 }`` *treated as successful and* ``{ ok: 0 }`` *as failed?*

The specification is consistent with what the server deems as a successful or failed command and
reports this as so. This also allows for server changes around this behaviour in the future to
require no change in the drivers to continue to be compliant.

The command listener API is responsible only for receiving and handling events sent from the lowest
level of the driver, and is only about informing listeners about what commands are sent and what
replies are received. As such, it would be innappropiate at this level for a driver to execute
custom logic around particular commands to determine what failure or success means for a particular
command. Implementators of the API are free to handle these events as they see fit, which may include
code that futher interprets replies to specific commands based on the presence or absence of other
fields in the reply beyond the ``ok`` field.

*2. Why are document sequences included as BSON arrays?*

The OP_MSG wire protocol was introduced in MongoDB 3.6, with document sequences as an optimization for bulk writes. We have chosen to represent these OP_MSGs as single command documents for now, until a need for a more accurate (and perhaps better-performing) command monitoring API for document sequences has been demonstrated.

*3. Why is BSON serialization and deserialization optional to include in durations?*

Different drivers will serialize and deserialize BSON at different levels of
the driver architecture.  For example, some parts of a command (e.g. inserted
document structs) could be pre-encoded early into a "raw" BSON form and the
final command with late additions like a session ID could encoded just before
putting it on the wire.

Rather than specify a duration rule that would be hard to satisfy consistently,
we allow duration to include BSON serialization/deserialization or not based on
the architecture needs of each driver.

--------
Security
--------

Some commands and replies will contain sensitive data and in order to not risk the leaking of this
data to external sources or logs their commands AND replies MUST be redacted from the events. The
value MUST be replaced with an empty BSON document. The list is as follows:

.. list-table::
   :header-rows: 1
   :widths: 50

   * - Command
   * - ``authenticate``
   * - ``saslStart``
   * - ``saslContinue``
   * - ``getnonce``
   * - ``createUser``
   * - ``updateUser``
   * - ``copydbgetnonce``
   * - ``copydbsaslstart``
   * - ``copydb``
   * - ``hello`` (or legacy hello) when ``speculativeAuthenticate`` is present

See the `MongoDB Handshake spec <https://github.com/mongodb/specifications/blob/master/source/mongodb-handshake/handshake.rst>`_
for more information on ``hello`` and legacy hello. Note that legacy hello has two different letter casings that must be taken
into account. See the previously mentioned MongoDB Handshake spec for details.

---
API
---

See the `Load Balancer Specification <../load-balancers/load-balancers.rst#events>`__ for details on the ``serviceId`` field.

.. code:: typescript

  interface CommandStartedEvent {

    /**
     * Returns the command.
     */
    command: Document;

    /**
     * Returns the database name.
     */
    databaseName: String;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the driver generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;

    /**
     * Returns the server connection id for the command. The server connection id is distinct from
     * the connection id and is returned by the hello or legacy hello response as "connectionId"
     * from the server on 4.2+. Drivers MAY use a wider type to represent the server connection ID
     * value, but the server's behavior is to return an Int32.
     */
    serverConnectionId: Optional<Int32>;

    /**
     * Returns the service id for the command when the driver is in load balancer mode.
     * For drivers that wish to include this in their ConnectionId object, this field is
     * optional.
     */
    serviceId: Optional<ObjectId>;
  }

  interface CommandSucceededEvent {

    /**
     * Returns the execution time of the event in the highest possible resolution for the platform.
     * The calculated value MUST be the time to send the message and receive the reply from the server
     * and MAY include BSON serialization and/or deserialization. The name can imply the units in which the
     * value is returned, i.e. durationMS, durationNanos.
     */
    duration: Int64;

    /**
     * Returns the command reply.
     */
    reply: Document;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the driver generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;

    /**
     * Returns the server connection id for the command. The server connection id is distinct from
     * the connection id and is returned by the hello or legacy hello response as "connectionId"
     * from the server on 4.2+. Drivers MAY use a wider type to represent the server connection ID
     * value, but the server's behavior is to return an Int32.
     */
    serverConnectionId: Optional<Int32>;

    /**
     * Returns the service id for the command when the driver is in load balancer mode.
     * For drivers that wish to include this in their ConnectionId object, this field is
     * optional.
     */
    serviceId: Optional<ObjectId>;
  }

  interface CommandFailedEvent {

    /**
     * Returns the execution time of the event in the highest possible resolution for the platform.
     * The calculated value MUST be the time to send the message and receive the reply from the server
     * and MAY include BSON serialization and/or deserialization. The name can imply the units in which the
     * value is returned, i.e. durationMS, durationNanos.
     */
    duration: Int64;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the failure. Based on the language, this SHOULD be a message string, exception
     * object, or error document.
     */
    failure: String,Exception,Document;

    /**
     * Returns the client generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;

    /**
     * Returns the server connection id for the command. The server connection id is distinct from
     * the connection id and is returned by the hello or legacy hello response as "connectionId"
     * from the server on 4.2+. Drivers MAY use a wider type to represent the server connection ID
     * value, but the server's behavior is to return an Int32.
     */
    serverConnectionId: Optional<Int32>;

    /**
     * Returns the service id for the command when the driver is in load balancer mode.
     * For drivers that wish to include this in their ConnectionId object, this field is
     * optional.
     */
    serviceId: Optional<ObjectId>;
  }


--------
Examples
--------

A Ruby subscriber to a query series and how it could handle it with respect to logging.

Ruby:

.. code:: ruby

  class LoggingSubscriber
    def logger
      Logger.new(STDERR)
    end

    def started(event)
      logger.info("COMMAND.#{event.command_name} #{event.address} STARTED: #{event.command.inspect}")
    end

    def succeeded(event)
      logger.info("COMMAND.#{event.command_name} #{event.address} COMPLETED: #{event.reply.inspect} (#{event.duration}s)")
    end

    def failed(event)
      logger.info("COMMAND.#{event.command_name} #{event.address} FAILED: #{event.message.inspect}: #{event.failure.inspect} (#{event.duration}s)")
    end
  end

  subscriber = LoggingSubscriber.new
  Mongo::Monitoring::Global.subscribe(Mongo::Monitoring::COMMAND, subscriber)

  # When the subscriber handles the events the log could show:
  # COMMAND.query 127.0.0.1:27017 STARTED: { $query: { name: 'testing' }}
  # COMMAND.query 127.0.0.1:27017 COMPLETED: { number_returned: 50 } (0.050s)


-------
Testing
-------

See the README in the test directory for requirements and guidance.


Changelog
=========

16 SEP 2015:
  - Removed ``limit`` from find test with options to support 3.2.
  - Changed find test read preference to ``primaryPreferred``.

1 OCT 2015:
  - Changed find test with a kill cursors to not run on server versions greater than 3.0
  - Added a find test with no kill cursors command which only runs on 3.1 and higher.
  - Added notes on which tests should run based on server versions.

19 OCT 2015:
  - Changed batchSize in the 3.2 find tests to expect the remaining value.

31 OCT 2015:
  - Changed find test on 3.1 and higher to ignore being run on sharded clusters.

22 NOV 2015:
  - Specify how to merge OP_MSG document sequences into command-started events.

29 MAR 2016:
  - Added note on guarantee of the request ids.

2 NOV 2016:
  - Added clause for not upconverting commands larger than maxBsonSize.

16 APR 2018:
  - Made inclusion of BSON serialization/deserialization in command durations
    to be optional.

12 FEB 2020:
  - Added legacy hello ``speculativeAuthenticate`` to the list of values that should be redacted.

15 APR 2021:
  - Added ``serviceId`` field to events.

5 MAY 2021:
  - Updated to use hello and legacy hello.

30 AUG 2021:
  - Added ``serverConnectionId`` field to ``CommandStartedEvent``, ``CommandSucceededEvent`` and
    ``CommandFailedEvent``.

18 MAY 2022:
  - Converted legacy tests to the unified test format.

2 SEPTEMBER 2022:
  - Remove material that only applies to MongoDB versions < 3.6.
