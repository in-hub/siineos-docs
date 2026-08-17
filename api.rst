================
SIINEOS HTTP API
================

This documentation describes the RESTful HTTP API for the **SMAC-Server** backend (SIINEOS). The API follows standard REST principles, using HTTP methods to perform actions on resources. All API routes are prefixed with ``/api/v1/``.

Overview
========

Most administrative and configuration-related routes require an authenticated session with **System Administrator** privileges, unless explicitly marked ``Public`` in an endpoint's Auth Level.

Authentication & Session Management
====================================

The system uses a dual-token scheme, with access tokens issued as JSON Web Tokens signed using the **EdDSA** algorithm.

.. list-table::
   :header-rows: 1
   :widths: 20 10 15 55

   * - Token
     - Type
     - Lifetime
     - Contents
   * - Access Token
     - JWT (EdDSA)
     - 3 minutes
     - User identity, roles, session identifier (``sid``)
   * - Refresh Token
     - UUID
     - 30 days
     - Used to obtain new access tokens without re-entering credentials

Authentication Requirement
---------------------------

For protected routes, the Access Token may be provided via any of the following methods:

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Method
     - Example
   * - **Authorization Header (Recommended)**
     - ``Authorization: Bearer <token>``
   * - **Custom Header**
     - ``x-access-token: <token>``
   * - **Query Parameter** (fallback)
     - ``?access_token=<token>``

Unless otherwise noted, all endpoints in this API require **System Administrator** privileges.

Endpoints
---------

1. Login
~~~~~~~~

Authenticates a user and initializes a new session.

- **URL:** ``/api/v1/auth/login``
- **Method:** ``POST``
- **Content-Type:** ``application/json``
- **Authentication Required:** None (Public)

**Request Body:**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Field
     - Type / Description
   * - ``username``
     - String (Required)
   * - ``password``
     - String (Required)

.. code-block:: json

   {
     "username": "string",
     "password": "string"
   }

**Success Response:**

- **Code:** ``200 OK``
- **Payload:**

  .. code-block:: json

     {
       "accessToken": "eyJhbGci...",
       "tokenType": "Bearer",
       "refreshToken": "uuid-string"
     }

**Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Status Code
     - Description / Payload
   * - ``200 OK``
     - ``{"accessToken": "...", "tokenType": "Bearer", "refreshToken": "..."}``
   * - ``400 Bad Request``
     - Invalid JSON, payload too large (>1000 bytes), or required fields missing.
   * - ``401 Unauthorized``
     - Invalid username or password.

2. Refresh Access Token
~~~~~~~~~~~~~~~~~~~~~~~

Uses a valid Refresh Token to generate a new, short-lived Access Token.

- **URL:** ``/api/v1/auth/refresh``
- **Method:** ``POST``
- **Content-Type:** ``application/json``
- **Authentication Required:** Refresh Token (in body)

**Request Body:**

.. code-block:: json

   {
     "refreshToken": "uuid-string"
   }

**Success Response:**

- **Code:** ``200 OK``
- **Payload:**

  .. code-block:: json

     {
       "accessToken": "eyJhbGci...",
       "tokenType": "Bearer"
     }

**Error Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Code
     - Description
   * - ``400 Bad Request``
     - Missing ``refreshToken`` in body, or payload exceeds 1000 bytes.
   * - ``401 Unauthorized``
     - Refresh token is invalid or has expired.

3. Logout
~~~~~~~~~

Invalidates the session associated with the provided Refresh Token.

- **URL:** ``/api/v1/auth/logout``
- **Method:** ``POST``
- **Content-Type:** ``application/json``
- **Authentication Required:** Refresh Token (in body)

**Request Body:**

.. code-block:: json

   {
     "refreshToken": "uuid-string"
   }

**Success Response:**

- **Code:** ``200 OK`` (no body)

**Error Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Code
     - Description
   * - ``400 Bad Request``
     - Payload exceeds 1000 bytes, or missing ``refreshToken``.
   * - ``401 Unauthorized``
     - Refresh token not found or invalid.

4. Close Session (Self-Service / Force Close)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Closes the specific session identified by the UUID in the URL.

**Security Constraint:** By default this is a self-service endpoint — the operation only succeeds if the ``sid`` (Session ID) claim inside the caller's current Access Token matches the ``sessionId`` provided in the URL path. Callers with elevated privileges may force-close sessions belonging to other users.

- **URL:** ``/api/v1/sessions/{sessionId}``
- **Method:** ``DELETE``
- **Authentication Required:** User Auth (``sid`` must match ``{sessionId}``, unless forcing)

**Path Parameters:**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Parameter
     - Type
     - Description
   * - ``sessionId``
     - ``string``
     - The unique UUID of the session to be terminated.

**Success Response:**

- **Code:** ``200 OK`` (no body)

**Error Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Code
     - Description
   * - ``401 Unauthorized``
     - No valid Access Token provided.
   * - ``403 Forbidden``
     - The Access Token is valid, but the ``sid`` claim does not match the requested ``sessionId``.
   * - ``404 Not Found``
     - The session was not found in the system.

Summary Table
-------------

.. list-table::
   :header-rows: 1
   :widths: 30 15 30 25

   * - Method & Endpoint
     - Method
     - Auth Level
     - Description
   * - ``POST /api/v1/auth/login``
     - ``POST``
     - Public
     - Authenticate and start a session.
   * - ``POST /api/v1/auth/refresh``
     - ``POST``
     - Public (Refresh Token in body)
     - Refresh access token using ``{"refreshToken": "..."}``.
   * - ``POST /api/v1/auth/logout``
     - ``POST``
     - Public (Refresh Token in body)
     - Log out using ``{"refreshToken": "..."}``.
   * - ``DELETE /api/v1/sessions/<sessionId>``
     - ``DELETE``
     - User Auth
     - Force/self close a specific session by ID.

Common API Patterns
====================

Most resource-based (collection) endpoints across this API follow these standard patterns, implemented via generic resource types. Sections below reference this pattern instead of re-describing it, and call out any deviations (e.g., Firewall's Move Up/Down).

Collection Operations
---------------------

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/<base_path>``
     - Retrieve all resources in the collection.
   * - ``POST``
     - ``/<base_path>``
     - Create a new resource. A UUID is automatically generated.
   * - ``PUT``
     - ``/<base_path>``
     - **Replace All:** Overwrite the entire collection.

Individual Resource Operations
------------------------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/<base_path>/{uuid}``
     - Retrieve details for a specific resource.
   * - ``PUT``
     - ``/<base_path>/{uuid}``
     - Update an existing resource by its UUID.
   * - ``PUT``
     - ``/<base_path>/disable/{uuid}``
     - Set the ``disabled`` status of a resource.
   * - ``DELETE``
     - ``/<base_path>/{uuid}``
     - Remove a resource from the collection.

Standard Error Codes
---------------------

The following status codes are common across resource endpoints in this API (Apps, Firewall, I/O, Alerting), unless otherwise noted in an endpoint's own section.

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - Code
     - Description
     - Context
   * - ``200 OK``
     - Request successful.
     - Returned for successful ``GET``, ``PUT``, ``DELETE``, and ``POST``.
   * - ``201 Created`` \*
     - Resource created.
     - *Note: current implementation returns* ``200 OK`` *with the new* ``uuid`` *in the body, rather than a true* ``201``.
   * - ``400 Bad Request``
     - Invalid input.
     - Payload exceeds size limit, or validation logic failed.
   * - ``401 Unauthorized``
     - No/invalid token.
     - Missing or invalid Access Token.
   * - ``403 Forbidden``
     - Permission denied.
     - Authenticated, but lacking permission for the resource.
   * - ``404 Not Found``
     - Resource missing.
     - The requested UUID does not exist in storage.
   * - ``409 Conflict``
     - Duplicate resource.
     - A resource-exists check returned true (e.g., on create).
   * - ``500 Internal Error``
     - Server error.
     - Error during data persistence or processing.

--------------

Apps Manager
============

The Apps Manager controls the lifecycle and diagnostic settings of applications running on the system. It handles application enablement, debugging, and tracing, and synchronizes state across the system using MQTT.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Endpoints
---------

1. Set App Control Property
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Updates a specific configuration property (``enabled``, ``debug``, or ``trace``) for a given application.

- **URL:** ``/api/v1/apps/{appId}/control/{controlProperty}``
- **Method:** ``PUT``
- **Auth Level:** SysAdmin

**Path Parameters:**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Parameter
     - Type
     - Description
   * - ``appId``
     - ``string``
     - The unique identifier of the application.
   * - ``controlProperty``
     - ``string``
     - The property to modify: ``enabled``, ``debug``, ``trace``.

**Request Body (JSON):**

.. code-block:: json

   true

*(The body must be a raw boolean value, e.g.* ``true`` *or* ``false`` *.)*

**Success Response:**

- **Code:** ``200 OK``

**Error Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Code
     - Description
   * - ``400 Bad Request``
     - Body exceeds 10 bytes, or is not a valid boolean.
   * - ``404 Not Found``
     - The requested ``controlProperty`` is not a valid control property.

Firewall Manager
================

The Firewall Manager provides a RESTful interface to manage network security policies, including incoming/outgoing traffic rules, IP forwarding, port forwarding, and internet connection sharing.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Resource Overview
-----------------

Each collection below supports the standard `Common API Patterns`_, plus **Move Up** / **Move Down** operations to change rule priority.

.. list-table::
   :header-rows: 1
   :widths: 25 40 35

   * - Resource
     - Base Path
     - Description
   * - Incoming Rules
     - ``/api/v1/firewall/rules/incoming``
     - Rules for traffic entering the system.
   * - Outgoing Rules
     - ``/api/v1/firewall/rules/outgoing``
     - Rules for traffic leaving the system.
   * - IP Forwarding Rules
     - ``/api/v1/firewall/rules/ipforwarding``
     - Rules for routing between specific interfaces.
   * - Port Forwarding Rules
     - ``/api/v1/firewall/rules/portforwarding``
     - NAT rules mapping external ports to internal IPs.

Rule Collection Operations
---------------------------

In addition to the standard `Collection Operations`_ and `Individual Resource Operations`_, all four rule resources above support:

.. list-table::
   :header-rows: 1
   :widths: 15 35 50

   * - Method
     - Endpoint
     - Description
   * - ``PUT``
     - ``/<base_path>/{uuid}/up``
     - Move a rule up in priority.
   * - ``PUT``
     - ``/<base_path>/{uuid}/down``
     - Move a rule down in priority.

Internet Connection Sharing (ICS)
-----------------------------------

Configures NAT/Masquerading from specific input interfaces to a designated output interface.

- **URL:** ``/api/v1/firewall/internet/sharing``
- **Methods:** ``GET`` (retrieve settings), ``POST`` (update settings)

**Request Body (for** ``POST`` **):**

.. code-block:: json

   {
     "enabled": true,
     "inputInterfaces": ["eth0", "wlan0"],
     "outputInterface": "eth1"
   }

**Success Response:**

- **Code:** ``200 OK``

**Error Responses:**

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Code
     - Description
   * - ``400 Bad Request``
     - Invalid data format, or unsupported ``outputInterface``.
   * - ``401 Unauthorized``
     - Admin privileges required.

Rule Property Reference
-------------------------

**Incoming / Outgoing Rules**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Property
     - Type
     - Description
   * - ``enabled``
     - ``bool``
     - Whether the rule is active.
   * - ``networkProtocol``
     - ``int``
     - Protocol number (e.g., ``6`` for TCP, ``17`` for UDP).
   * - ``inputInterface``
     - ``string``
     - Interface for incoming traffic (Incoming rules only).
   * - ``outputInterface``
     - ``string``
     - Interface for outgoing traffic (Outgoing rules only).
   * - ``sourceAddress``
     - ``string``
     - Source IP/CIDR.
   * - ``destinationAddress``
     - ``string``
     - Destination IP/CIDR.
   * - ``destinationPorts``
     - ``string``
     - Space-separated list of ports (e.g., ``"80 443"``).
   * - ``action``
     - ``int``
     - The nftables action (``1`` = Accept, ``2`` = Drop).

**IP Forwarding Rules**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Property
     - Type
     - Description
   * - ``inputInterface``
     - ``string``
     - Source interface.
   * - ``outputInterface``
     - ``string``
     - Destination interface.
   * - ``sourceAddress``
     - ``string``
     - Source IP/CIDR.
   * - ``destinationAddress``
     - ``string``
     - Destination IP/CIDR.

**Port Forwarding Rules**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Property
     - Type
     - Description
   * - ``localPort``
     - ``int``
     - Port on the local machine.
   * - ``destinationAddress``
     - ``string``
     - Target internal IP.
   * - ``destinationPort``
     - ``int``
     - Target internal port.
   * - ``networkProtocol``
     - ``int``
     - Protocol number.
   * - ``masquerade``
     - ``bool``
     - Whether to perform SNAT (default: ``false``).

Summary Table
-------------

.. list-table::
   :header-rows: 1
   :widths: 45 25 30

   * - Endpoint
     - Method
     - Description
   * - ``/api/v1/firewall/rules/incoming``
     - ``GET/POST/PUT/DELETE``
     - Manage Incoming rules.
   * - ``/api/v1/firewall/rules/outgoing``
     - ``GET/POST/PUT/DELETE``
     - Manage Outgoing rules.
   * - ``/api/v1/firewall/rules/ipforwarding``
     - ``GET/POST/PUT/DELETE``
     - Manage IP Forwarding rules.
   * - ``/api/v1/firewall/rules/portforwarding``
     - ``GET/POST/PUT/DELETE``
     - Manage Port Forwarding rules.
   * - ``/api/v1/firewall/internet/sharing``
     - ``GET/POST``
     - Manage Internet Connection Sharing.

I/O Manager
===========

The I/O Manager manages the lifecycle and configuration of all Input/Output devices (``IoUnits``) and the logical connections between their signals. It supports multiple providers (Modbus, MQTT, OPC UA, Synthetic, etc.) and provides a hierarchical configuration model.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Device & Unit Management
------------------------

.. list-table::
   :header-rows: 1
   :widths: 12 45 43

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/api/v1/io/units``
     - Retrieve all active ``IoUnits``.
   * - ``POST``
     - ``/api/v1/io/units``
     - **Add Unit:** Create a new unit (Payload: ``{"providerId": "..."}``). Returns the new ``uuid``.
   * - ``DELETE``
     - ``/api/v1/io/units/{uuid}``
     - Remove a unit and its configuration.
   * - ``GET``
     - ``/api/v1/io/units/{uuid}/config``
     - Retrieve the full configuration object of a unit.
   * - ``PUT``
     - ``/api/v1/io/units/{uuid}/config``
     - Overwrite the entire configuration of a unit.
   * - ``POST``
     - ``/api/v1/io/units/{uuid}/config/{objId}``
     - Update a specific sub-object in the unit's configuration.
   * - ``PUT``
     - ``/api/v1/io/units/{uuid}/name``
     - Set the display name of the unit.
   * - ``PUT``
     - ``/api/v1/io/units/{uuid}/upstream``
     - Set upstream unit and port for signal chaining.
   * - ``GET``
     - ``/api/v1/io/units/{uuid}/icon``
     - Download the unit's custom icon (Binary).
   * - ``PUT``
     - ``/api/v1/io/units/{uuid}/icon``
     - Upload a custom icon (Binary).
   * - ``DELETE``
     - ``/api/v1/io/units/{uuid}/icon``
     - Clear the custom icon.
   * - ``POST``
     - ``/api/v1/io/units/{uuid}/control/{action}``
     - Execute a control command on a unit or specific signal (e.g., ``load-modbus-device-profile``).

Signal Management
-----------------

All signal endpoints below are scoped under a unit: ``/api/v1/io/units/{unitUuid}/signals...``

.. list-table::
   :header-rows: 1
   :widths: 12 30 58

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/``
     - Retrieve all signals belonging to the unit.
   * - ``POST``
     - ``/``
     - Add a single new signal to the unit.
   * - ``PUT``
     - ``/``
     - Bulk add/replace signals (Payload: ``Array<SignalProperties>``).
   * - ``GET``
     - ``/{uuid}``
     - Retrieve details for a specific signal.
   * - ``PUT``
     - ``/{uuid}``
     - Update properties of a specific signal.
   * - ``DELETE``
     - ``/?uuids=id1,id2``
     - Remove multiple signals via comma-separated query parameter.
   * - ``PUT``
     - ``/config``
     - Batch/bulk update configuration for multiple signals.
   * - ``PUT``
     - ``/{signalUuid}/config``
     - Update the processing configuration for a specific signal (see `5. Connection & Signal Processing`_).
   * - ``POST``
     - ``/{uuid}/duplicate``
     - Duplicate a signal (Payload: ``{"name": "New Name"}``).
   * - ``POST``
     - ``/reset``
     - Reset signals to default values (Payload: ``Array<uuid>``).

Synthetic Signals
-----------------

The Synthetic Signals provider allows creation of "virtual" signals that do not correspond to physical hardware, useful for calculations or testing.

**Base Path:** ``/api/v1/io/synthetic/signals``

.. list-table::
   :header-rows: 1
   :widths: 12 30 58

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/``
     - Retrieve all synthetic signals.
   * - ``POST``
     - ``/``
     - Create new synthetic signal(s) (Payload: ``Array<SignalProperties>``).
   * - ``PUT``
     - ``/``
     - Replace all synthetic signal configurations.
   * - ``GET``
     - ``/{uuid}``
     - Retrieve a synthetic signal's details.
   * - ``POST``
     - ``/{uuid}/duplicate``
     - Duplicate a synthetic signal.
   * - ``POST``
     - ``/<uuid>/clone/<uuid>``
     - Clone settings from one synthetic signal onto another.
   * - ``POST``
     - ``/{uuid}/name/propagate``
     - Sync/copy name from the synthetic signal to its linked (source) I/O signal.
   * - ``POST``
     - ``/{uuid}/name/update``
     - Sync the synthetic signal's name from its actual source I/O signal.

**Synthetic Calculation Capabilities**

Synthetic signals can perform operations using variables ``A`` (Source 1), ``B`` (Source 2), and ``P`` (Previous Value):

- **Basic Math:** ``+``, ``-``, ``*``, ``/``
- **Logic:** ``&&`` (AND), ``||`` (OR)
- **Boolean Logic:** ``..`` (result is ``1`` if A is truthy and B is falsy), ``+.`` (sets current value to A), ``++`` (increments by A).
- **Expressions:** Mathematical formulas (e.g., ``1.25 + (x / 2)``) via expression evaluation.

Modbus Endpoint Management
--------------------------

Configures the Modbus endpoint settings which allows other hardware to read/write mapped I/O signals.

**Base Path:** ``/api/v1/io/endpoints/modbus``

.. list-table::
   :header-rows: 1
   :widths: 15 20 65

   * - Method
     - Endpoint
     - Description
   * - ``GET``
     - ``/tcp``
     - Get current Modbus TCP settings.
   * - ``PUT``
     - ``/tcp``
     - Update Modbus TCP settings.
   * - ``GET``
     - ``/rtu``
     - Get current Modbus RTU settings.
   * - ``PUT``
     - ``/rtu``
     - Update Modbus RTU settings.

**TCP Configuration Payload Example:**

.. code-block:: json

   {
     "enabled": true,
     "networkAddress": "192.168.1.50",
     "networkPort": 502,
     "tcpPacketFlowOptimization": 0,
     "requestTimeout": 500,
     "requestRetryCount": 2,
     "requestQueueSizeLimit": 500,
     "interFrameDelay": 0,
     "maximumBlockGap": 0,
     "maximumBlockLength": 128
   }

Connection & Signal Processing
------------------------------

**Logical Connections**

Manages the routing of data from one signal to another.

- **URL:** ``/api/v1/io/connections``
- **Operations:** ``GET``, ``POST``, ``PUT`` (Replace All), ``PUT`` (Update Single), ``DELETE``

**Signal Processing**

Configures the mathematical and logical transformations applied to a signal as it passes through a connection.

- **URL:** ``/api/v1/io/units/{unitUuid}/signals/{signalUuid}/config``
- **Method:** ``PUT``

Summary Table
-------------

.. list-table::
   :header-rows: 1
   :widths: 40 15 45

   * - Method & Endpoint
     - Auth Level
     - Description
   * - ``POST /api/v1/io/units``
     - SysAdmin
     - Add a new I/O unit (Payload: ``{"providerId": "..."}``).
   * - ``DELETE /api/v1/io/units/<uuid>``
     - SysAdmin
     - Remove a unit.
   * - ``GET /api/v1/io/units/<uuid>/config``
     - SysAdmin
     - Get full configuration of a unit.
   * - ``PUT /api/v1/io/units/<uuid>/config``
     - SysAdmin
     - Update unit configuration.
   * - ``POST /api/v1/io/units/<uuid>/control/<action>``
     - SysAdmin
     - Execute a command on a unit or specific signal.
   * - ``POST /api/v1/io/units/<uuid>/signals``
     - SysAdmin
     - Add a new signal to a unit.
   * - ``PUT /api/v1/io/units/<uuid>/signals/config``
     - SysAdmin
     - Batch update signal configurations.
   * - ``POST /api/v1/io/synthetic/signals/<uuid>/clone/<uuid>``
     - SysAdmin
     - Clone settings from one synthetic signal to another.
   * - ``POST /api/v1/io/synthetic/signals/<uuid>/name/propagate``
     - SysAdmin
     - Sync name from a synthetic signal to its source.
   * - ``POST /api/v1/io/synthetic/signals/<uuid>/name/update``
     - SysAdmin
     - Sync name from a signal back to the synthetic definition.

Alerting Manager
=================

The Alerting Manager provides a RESTful interface to manage three primary resources: **Alert Signals**, **Alert Destinations**, and **Alert Rules**.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Resource Overview
-------------------

.. list-table::
   :header-rows: 1
   :widths: 25 35 40

   * - Resource
     - Base Path
     - Description
   * - Alert Signals
     - ``/api/v1/alerting/signals``
     - Configuration for incoming alert triggers.
   * - Alert Destinations
     - ``/api/v1/alerting/destinations``
     - Target endpoints (Email, MQTT, etc.).
   * - Alert Rules
     - ``/api/v1/alerting/rules``
     - Logic connecting Signals to Destinations.

All three resources inherit from the same base REST implementation and follow the standard `Collection Operations`_ and `Individual Resource Operations`_ patterns. The table below gives the per-resource payload for each operation.

Endpoint Detail (applies to Signals, Destinations, and Rules)
-----------------------------------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 12 20 33 35

   * - Method
     - Endpoint
     - Payload (JSON)
     - Notes
   * - ``GET``
     - ``/``
     - None
     - Returns list of all resources.
   * - ``POST``
     - ``/``
     - ``{ ...resourceData }``
     - Generates a new ``uuid``.
   * - ``GET``
     - ``/{uuid}``
     - None
     - Returns a single resource.
   * - ``PUT``
     - ``/{uuid}``
     - ``{ ...resourceData }``
     - Updates the specific resource.
   * - ``PUT``
     - ``/disable/{uuid}``
     - ``{ "disabled": bool }``
     - Toggles resource status.
   * - ``DELETE``
     - ``/{uuid}``
     - None
     - Deletes the resource.

*Note:* ``resourceData`` *refers to* ``signalData``, ``destinationData``, *or* ``ruleData`` *respectively, depending on the resource being addressed.*

Error Codes
-------------

See `Standard Error Codes`_. No resource-specific deviations apply.

--------------

System Manager
==============

Provides read-only diagnostic information as well as SysAdmin-level system control (log retrieval, reboot, clock management).

**Authentication:** Mixed — see Auth Level per endpoint below.

System & Diagnostics
---------------------

.. list-table::
   :header-rows: 1
   :widths: 40 15 45

   * - Method & Endpoint
     - Auth Level
     - Description
   * - ``GET /api/v1/system/information``
     - Public
     - Get hostname, uptime, OS version, and IP addresses.
   * - ``GET /api/v1/system/metrics``
     - Public
     - Get CPU, memory, storage, and network traffic stats.
   * - ``GET /api/v1/system/journal/recent``
     - SysAdmin
     - Retrieve recent system log messages.
   * - ``POST /api/v1/system/reboot``
     - SysAdmin
     - Reboot the device.
   * - ``POST /api/v1/system/system-clock``
     - SysAdmin
     - Set the system hardware clock.

Time Series Database (VictoriaMetrics)
=======================================

Manages historical metrics data stored in VictoriaMetrics, including querying, CSV export, and deletion of time series.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Metrics & Export
-----------------

.. list-table::
   :header-rows: 1
   :widths: 40 15 45

   * - Method & Endpoint
     - Auth Level
     - Description
   * - ``GET /api/v1/vmetrics/timeseries``
     - SysAdmin
     - Retrieve metric labels for a time range.
   * - ``POST /api/v1/vmetrics/export/csv``
     - SysAdmin
     - Export metric data to a CSV file (supports rollups).
   * - ``POST /api/v1/vmetrics/timeseries/delete``
     - SysAdmin
     - Delete specific time series matching provided IDs.

System Update Manager
======================

Handles chunked firmware/update-bundle uploads and reports on installation progress.

**Authentication:** System Administrator privileges required (see `Authentication Requirement`_).

Firmware Updates
-----------------

.. list-table::
   :header-rows: 1
   :widths: 40 15 45

   * - Method & Endpoint
     - Auth Level
     - Description
   * - ``POST /api/v1/system/update/upload/start``
     - SysAdmin
     - Initialize a new update file upload.
   * - ``POST /api/v1/system/update/upload/chunk``
     - SysAdmin
     - Upload a chunk of the update bundle.
   * - ``GET /api/v1/system/update/install/state``
     - SysAdmin
     - Get progress and status of the current installation.
