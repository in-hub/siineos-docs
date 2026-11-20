SIINEOS System Bus specification
================================

General
-------

- provided by internal MQTT broker
- SMAC server has full access
- read-only access publicly available if enabled in :guilabel:`System` → :guilabel:`Service` configuration page
- access from Docker-based apps such as NodeRED etc. through the DNS alias ``host.docker.internal`` (pass ``--add-host=host.docker.internal:host-gateway`` when running Docker container manually)

System
------

Device
^^^^^^

- ``device/info/deviceName`` - descriptive user-defined device name
- ``device/info/deviceType`` - type/hardware model of the local device as defined in the InCore System.DeviceType enumeration
- ``device/info/deviceTypeName`` - type/hardware model of the local device
- ``device/info/deviceSubtype`` - model-specific sub type specification
- ``device/info/hostname`` - the user-defined hostname of the local device
- ``device/info/location`` - the user-defined location of the local device

Settings
^^^^^^^^

- ``system/settings/timezone``

Networking
^^^^^^^^^^

- common prefix: ``system/networking/``
- each network interface publishes its status information under ``system/networking/interfaces/<IFNAME>/status`` as a JSON object:

  - ``operationalState`` - operational state as defined in the InCore NetworkInterface.OperationalState enumeration
  - ``setupState`` - setup state as defined in the InCore NetworkInterface.SetupState enumeration
  - ``hardwareAddress`` - hardware/MAC address
  - ``networkAddresses`` - JSON array with IP addresses

I/O units
---------

- common prefix: ``iounits/``
- global I/O units announce topic: ``iounits/announce`` with a JSON array containing the instance UUIDs of each I/O unit
- global I/O providers info topic: ``iounits/providers`` with a JSON array containing meta data for every available I/O provider
- each I/O unit announces itself under ``iounits/<UNIT-UUID>/announce`` as a JSON object containing state and meta data describing the unit instance:

  - ``providerId`` - internal identifier of the instantiated I/O provider
  - ``providerName`` - display name of the instantiated I/O provider
  - ``providerType`` - type of the instantiated I/O provider (enum index of: Device, Sensor, FieldbusConnector, FieldbusNode, VirtualProvider)
  - ``providerConnectivity`` - specification of how the I/O unit is connected to the system (enum index of: Direct, AnalogInterface, DigitalInterface, BackplaneBus, USB, RS485, Network, FieldbusPort, CanBus)
  - ``uuid`` - the UUID identifying the instance
  - ``address`` - an optional address such as port name or IP address which the I/O unit connects to
  - ``name`` - user-defined display name
  - ``location`` - user-defined location
  - ``enabled`` - user-defined state
  - ``connected`` - indicates whether the I/O unit is connected (depending on the connectivity type)
  - ``error`` - error message if the I/O unit is enabled and in an error state, otherwise empty
  - ``icon`` - the URL of the icon; for user-defined icons the URL of the HTTP API endpoint ``/api/v1/io/units/<UNIT-UUID>/icon`` with a hash of the icon data as query string
  - ``busy`` - indicates whether the I/O unit is busy, e.g. signal discovery etc.
  - ``signalsBrowsable`` - indicates whether signals can be discovered/browsed by the I/O unit
  - ``signalsResettable`` - indicates whether signals can be reset (e.g. counters)
  - ``signalAddressesContainPinAssignment`` - indicates whether the addresses of the signals describe a pin assignment
  - ``signals`` - JSON array of I/O signal UUIDs
  - ``upstreamUnit`` - UUID of the I/O unit which this I/O unit is attached to (empty if none)
  - ``upstreamPort`` - UUID of the port of the upstream I/O unit which this I/O unit is attached to (empty if none)
  - ``ports`` - JSON array with the UUIDs of the ports provided by the I/O unit

I/O ports
^^^^^^^^^

- each port of an I/O unit announces itself under ``iounits/<UNIT-UUID>/ports/<PORT-UUID>/announce`` as a JSON object:

  - ``uuid`` - the UUID identifying the port
  - ``type`` - port type (I/O units can be attached to ports matching the ``compatiblePortType`` of their I/O provider)
  - ``identifier``
  - ``name``
  - ``address``
  - ``status`` - ``ok``, ``inactive`` or ``error``
  - ``editable`` - indicates whether the port has settings intended to be changed by the user
  - ``attachedDevice`` - description of the device attached to the port, if known

I/O signals
^^^^^^^^^^^

- each I/O signal of a I/O unit announces itself under ``iounits/<UNIT-UUID>/signals/<SIGNAL-ID>/announce`` as a JSON object containing meta data describing the signal instance:

  - ``uuid``
  - ``type``
  - ``identifier``
  - ``enabled``
  - ``address``
  - ``name``
  - ``group``
  - ``dataSeriesSet``
  - ``readable``
  - ``writable``
  - ``dataType``
  - ``unit``
  - ``decimals``
  - ``minValue``
  - ``maxValue``
  - ``color``
  - ``visualization``

- ``iounits/<UNIT-UUID>/signals/<SIGNAL-ID>/value`` - current signal as read from or written to the signal (read-only)
- ``iounits/<UNIT-UUID>/signals/<SIGNAL-ID>/timestamp`` - timestamp referring to the last signal read/write

Alert signals
-------------

- each alert signal announces itself under ``alerting/signals/<SIGNAL-ID>/announce`` as a JSON object containing meta data describing the signal instance:

  - ``uuid``
  - ``name``
  - ``disabled``
  - ``evalMode``
  - ``evalParams``
  - ``cycleTime``
  - ``stateTransition``
  - ``severity``
  - ``category``
  - ``source``

- ``alerting/signals/<SIGNAL-ID>/state`` - current state of the alert signal (0=OK, 1=ALARM)
- ``alerting/signals/<SIGNAL-ID>/timestamp`` - timestamp referring to the last state change

Apps
----

- common prefix: ``apps/``
- every app announces itself under ``apps/<APPNAME>/`` with various meta data topics:

  - ``id``
  - ``name``
  - ``version``
  - ``vendor``
  - ``homepage``
  - ``iconUrl``
  - ``appUrl``
  - ``adminUrl``
  - ``description``
  - ``alertTriggerUrl``
  - ``licensing``

- state of app (enabled/disabled) can be controlled through the ``apps/<APPNAME>/enabled`` topic
- debug message logging of app (enabled/disabled) can be controlled through the ``apps/<APPNAME>/debug`` topic (SIINEOS 2.8+)
- trace message logging of app (enabled/disabled) can be controlled through the ``apps/<APPNAME>/trace`` topic (SIINEOS 2.8+)

