SIINEOS HTTP API reference
==========================

Overview
--------

This document describes the HTTP REST API exposed by the SMAC-Server application.
The API is served on **port 80** (HTTP) and **port 443** (HTTPS, when a TLS
certificate is configured). All endpoints are prefixed with ``/api/v1``.

The following table lists the top-level API areas. Each area is documented in
its own section below.

.. list-table:: General endpoint summary
   :header-rows: 1

   * - Area
     - Base Path
     - Description
   * - Authentication
     - ``/api/v1/auth``, ``/api/v1/sessions``
     - Login, token refresh, logout, session management → :ref:`authentication`
   * - Users
     - ``/api/v1/users``
     - User account CRUD and disable → :ref:`users`
   * - System
     - ``/api/v1/system``, ``/api/v1/storage``, ``/api/v1/sms``
     - Device info, metrics, networking, clock, mail, SMS, config → :ref:`system`
   * - System Update
     - ``/api/v1/system/update``
     - OTA update upload and installation → :ref:`system-update`
   * - Time-Series Database
     - ``/api/v1/vmetrics``
     - VictoriaMetrics query, export, delete, reset → :ref:`timeseries`
   * - I/O Management
     - ``/api/v1/io``
     - Units, signals, ports, connections, processing chains → :ref:`io-management`
   * - Synthetic Signals
     - ``/api/v1/io/synthetic``
     - Virtual computed signals → :ref:`synthetic-signals`
   * - Modbus Server
     - ``/api/v1/io/endpoints/modbus``
     - Built-in Modbus TCP/RTU server and registers → :ref:`modbus-server`
   * - Firewall
     - ``/api/v1/firewall``
     - nftables rules and internet sharing → :ref:`firewall`
   * - Alerting
     - ``/api/v1/alerting``
     - Alert signals, destinations, and rules → :ref:`alerting`
   * - Licensing
     - ``/api/v1/licensing``
     - License import, list, and removal → :ref:`licensing`
   * - Apps
     - ``/api/v1/apps``
     - Application enable/disable and debug control → :ref:`apps`
   * - Ping / Health
     - ``/api/v1/ping``
     - Liveness check → :ref:`ping`

.. _authentication:

Authentication & Authorization
------------------------------

The API uses **JSON Web Tokens (JWT)** signed with the **EdDSA** algorithm.
Two token types are issued:

- **Access Token** – short-lived (3 minutes), used to authorize API requests.
- **Refresh Token** – long-lived (30 days), used to obtain a new access token
  without re-authenticating.

Token Claims
~~~~~~~~~~~~

The access token contains the following claims:

- ``sub`` – the login name of the authenticated user (string)
- ``iat`` – issued-at timestamp (ISO 8601)
- ``exp`` – expiry timestamp (ISO 8601, 3 minutes after ``iat``)
- ``sid`` – session identifier (UUID without dashes)
- ``roles`` – array of role name strings, e.g. ``["SystemAdministrator"]``

Available Roles
~~~~~~~~~~~~~~~

- ``SystemAdministrator`` – full access to all endpoints
- ``GlobalAppAdministrator`` – application-level administration
- ``AppAdministrator`` – per-application administration
- ``AppUser`` – standard user access

Authorization Header
~~~~~~~~~~~~~~~~~~~~

Authenticated requests must carry the access token in one of the following ways
(listed in order of preference by the server):

1. ``Authorization: Bearer <accessToken>``
2. ``x-access-token: <accessToken>`` (custom header)
3. ``?access_token=<accessToken>`` (query parameter, used by
   :func:`hasSystemAdministratorAccessToken` for certain routes)

Most endpoints require the **SystemAdministrator** role. A small number of
endpoints (ping, system information, system metrics, system clock read,
licensing list, unit icon retrieval) are accessible without authentication.

Session Lifecycle
~~~~~~~~~~~~~~~~~

1. **Login** – exchange credentials for an access + refresh token pair.
2. **Refresh** – exchange a refresh token for a new access token.
3. **Logout** – invalidate the refresh token (and thus the session).
4. **Close Session** – delete a specific session by its ID (requires a valid
   access token belonging to that session).

Endpoints
~~~~~~~~~

POST /api/v1/auth/login
^^^^^^^^^^^^^^^^^^^^^^^^

Authenticate with username and password.

- **Authentication:** none
- **Request body** (JSON, max 1000 bytes):

  - ``username`` (string, required) – login name
  - ``password`` (string, required) – plaintext password

  Example:

  .. code-block:: json

    {
      "username": "hubadmin",
      "password": "hubadmin"
    }

- **Responses:**

  - ``200 OK`` – body is JSON:

    .. code-block:: json

      {
        "accessToken": "eyJhbGciOiJFZERTQSIs...",
        "tokenType": "Bearer",
        "refreshToken": "a1b2c3d4e5f6..."
      }

  - ``400 Bad Request`` – body too large or missing fields
  - ``401 Unauthorized`` – invalid credentials

POST /api/v1/auth/refresh
^^^^^^^^^^^^^^^^^^^^^^^^^^

Obtain a new access token using a refresh token.

- **Authentication:** none
- **Request body** (JSON, max 1000 bytes):

  - ``refreshToken`` (string, required)

  Example:

  .. code-block:: json

    {
      "refreshToken": "a1b2c3d4e5f6..."
    }

- **Responses:**

  - ``200 OK`` – body is JSON:

    .. code-block:: json

      {
        "accessToken": "eyJhbGciOiJFZERTQSIs...",
        "tokenType": "Bearer"
      }

  - ``400 Bad Request`` – body too large or missing field
  - ``401 Unauthorized`` – unknown or expired refresh token
  - ``500 Internal Server Error`` – token creation failed

POST /api/v1/auth/logout
^^^^^^^^^^^^^^^^^^^^^^^^^

Invalidate a session by its refresh token.

- **Authentication:** none
- **Request body** (JSON, max 1000 bytes):

  - ``refreshToken`` (string, required)

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large or missing field
  - ``401 Unauthorized`` – unknown refresh token

DELETE /api/v1/sessions/{sessionId}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Close a specific session. The caller's access token must belong to the same
session (``sid`` claim must match ``sessionId``).

- **Authentication:** valid access token (any role), ``sid`` must match
- **Path parameter:**

  - ``sessionId`` (string) – UUID of the session to close

- **Responses:**

  - ``200 OK`` – empty body
  - ``401 Unauthorized`` – missing or invalid token
  - ``403 Forbidden`` – token valid but ``sid`` does not match
  - ``404 Not Found`` – session not found


Users
-----

.. _users:

User management is exposed as a standard REST collection. The collection is a
**plain JSON array of objects** (no ``"data"`` sub-key wrapper).

User Object Schema
~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required) – unique identifier (auto-generated on create)
- ``loginName`` (string, required) – unique login name
- ``fullName`` (string, required) – display name
- ``password`` (string, required on create, optional on modify) – plaintext
  password; stored as SHA-256 hash server-side. **Never returned in GET
  responses.**
- ``role`` (integer, required) – one of:

  - ``0`` – SystemAdministrator
  - ``1`` – GlobalAppAdministrator
  - ``2`` – AppAdministrator
  - ``3`` – AppUser

- ``disabled`` (boolean, optional) – whether the account is disabled
  (defaults to ``false`` on create)

Endpoints
~~~~~~~~~

GET /api/v1/users
^^^^^^^^^^^^^^^^^^^

List all users (passwords are stripped).

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON array:

  .. code-block:: json

    [
      {
        "uuid": "907cc14b665a47f2963907f344f7bb73",
        "loginName": "hubadmin",
        "fullName": "HUB Administrator",
        "role": 0,
        "disabled": false
      }
    ]

GET /api/v1/users/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve a single user.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``uuid`` (string)
- **Response ``200 OK``** – single user object (no password)
- **Response ``404 Not Found``** – user not found

POST /api/v1/users
^^^^^^^^^^^^^^^^^^^^

Create a new user.

- **Authentication:** SystemAdministrator
- **Request body** (JSON):

  .. code-block:: json

    {
      "loginName": "operator1",
      "fullName": "Operator One",
      "password": "s3cret!",
      "role": 3
    }

- **Responses:**

  - ``200 OK`` – body is the new user's UUID (string)
  - ``400 Bad Request`` – invalid body
  - ``409 Conflict`` – a user with the same ``loginName`` already exists

PUT /api/v1/users/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Modify an existing user.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``uuid`` (string)
- **Request body** (JSON):

  .. code-block:: json

    {
      "loginName": "operator1",
      "fullName": "Operator One (renamed)",
      "password": "newPass123",
      "passwordConfirm": "newPass123",
      "role": 3,
      "disabled": false
    }

  Notes:

  - ``password`` is optional. If omitted or empty, the existing password hash
    is preserved.
  - ``passwordConfirm`` is accepted but stripped before storage.
  - If the user is not found and no password is supplied, the request fails.

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid body
  - ``404 Not Found`` – user not found

DELETE /api/v1/users/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Delete a user. The built-in ``hubadmin`` account cannot be deleted.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``uuid`` (string)
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – attempt to delete the protected ``hubadmin`` account
  - ``404 Not Found`` – user not found

PUT /api/v1/users/disable/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Enable or disable a user account.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``uuid`` (string)
- **Request body** (JSON):

  .. code-block:: json

    {
      "disabled": true
    }

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid body
  - ``404 Not Found`` – user not found


.. _system:

System
------

System-level endpoints cover device information, metrics, networking, storage,
clock, mail, SMS, and configuration objects.

Endpoints
~~~~~~~~~

GET /api/v1/system/information
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve basic system information.

- **Authentication:** none
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "systemClock": 1721560000,
      "uptime": 86400,
      "osVersion": "1.2.3 (Stable)",
      "osLicenseValid": 1,
      "osLicenseExpiryDate": "2027-01-01",
      "ipAddresses": [
        "192.168.1.10 (eth0)",
        "10.0.0.5 (wlan0)",
        "192.168.123.1 (usb0)"
      ]
    }

GET /api/v1/system/metrics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve real-time system metrics.

- **Authentication:** none
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "totalCpuUsage": 23.5,
      "brokenDownCpuUsage": { "user": 10.2, "system": 5.1, "idle": 84.7 },
      "systemLoad": 0.42,
      "memoryTotal": 2097152,
      "memoryAvailable": 1572864,
      "storageTotal": 16106127360,
      "storageAvailable": 12884901888,
      "storageBytesReadPerSecond": 1024,
      "storageBytesWrittenPerSecond": 2048,
      "eth0Rx": 1500000,
      "eth0Tx": 800000,
      "eth1Rx": 0,
      "eth1Tx": 0,
      "wlan0Rx": 500000,
      "wlan0Tx": 300000,
      "wwan0Rx": 0,
      "wwan0Tx": 0
    }

GET /api/v1/system/journal/recent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve the most recent system journal messages.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – plain-text, one message per line

GET /api/v1/system/processes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

List running system processes.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON array of process objects

GET /api/v1/system/networking/available-wireless-networks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Scan for available Wi-Fi networks.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON array of network objects

GET /api/v1/system/clock
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read the current system clock (seconds since epoch).

- **Authentication:** none
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    1721560000

POST /api/v1/system/clock
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Set the system clock.

- **Authentication:** SystemAdministrator
- **Request body** – plain-text Unix timestamp (seconds), max 128 bytes.
  Value must be greater than ``1234567890``.

  Example: ``1721560000``

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large or value out of range

POST /api/v1/system/hardware-clocks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Synchronise hardware clocks to the current system time.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – empty body

POST /api/v1/system/openvpn/client/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write an OpenVPN client configuration file and restart the service.

- **Authentication:** SystemAdministrator
- **Request body** – raw OpenVPN config text, max 64 KiB
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large
  - ``500 Internal Server Error`` – failed to write file

GET /api/v1/storage/usage
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve storage usage details.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON object with storage partition details

POST /api/v1/storage/docker/cleanup
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove all Docker files except volumes.

- **Authentication:** SystemAdministrator
- **Responses:**

  - ``200 OK`` – empty body
  - ``500 Internal Server Error`` – cleanup failed

POST /api/v1/storage/docker/reset
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove all Docker files including volumes.

- **Authentication:** SystemAdministrator
- **Responses:**

  - ``200 OK`` – empty body
  - ``500 Internal Server Error`` – reset failed

DELETE /api/v1/storage/docker/volumes/{name}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove a specific Docker volume.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``name`` (string) – volume name
- **Responses:**

  - ``200 OK`` – body is the volume name
  - ``404 Not Found`` – volume not found

POST /api/v1/system/device/identify
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Trigger the device identification LED blink sequence (10 seconds).

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – empty body

POST /api/v1/system/reboot
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Reboot the device.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – empty body

POST /api/v1/system/mail/test
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Send a test e-mail using the supplied SMTP settings (does not persist them).

- **Authentication:** SystemAdministrator
- **Request body** (JSON):

  .. code-block:: json

    {
      "serverAddress": "smtp.example.com",
      "serverPort": 587,
      "encryptionMode": 2,
      "authenticationMethod": 2,
      "username": "mailer@example.com",
      "password": "secret",
      "senderName": "SIINEOS Hub",
      "senderAddress": "hub@example.com",
      "recipient": "admin@example.com",
      "subject": "Test",
      "body": "This is a test e-mail."
    }

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – e-mail could not be sent

POST /api/v1/sms
^^^^^^^^^^^^^^^^^^

Send an SMS message via the cellular modem.

- **Authentication:** SystemAdministrator
- **Request body** (JSON, max 4 KiB):

  .. code-block:: json

    {
      "recipientNumbers": "+4915112345678,+491529876543",
      "text": "Alert: Temperature high on Unit 3"
    }

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – missing fields or body too large
  - ``500 Internal Server Error`` – SMS send failed

System Configuration Objects
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

System settings are exposed as **configuration objects**. In the JSON payload,
each property is wrapped in a ``"data"`` sub-key. For example:

.. code-block:: json

  {
    "hostname": { "data": "hub-gm100-abc123" },
    "deviceName": { "data": "Central Gateway" },
    "location": { "data": "Building 1, Room 234" }
  }

The following configuration objects are available:

- ``GET/POST /api/v1/system/config/datetime`` – timezone, NTP server, browser
  auto-sync
- ``GET/POST /api/v1/system/config/eth0`` – Ethernet 1 settings (mode, IP,
  DHCP server, etc.)
- ``GET/POST /api/v1/system/config/eth1`` – Ethernet 2 settings
- ``GET/POST /api/v1/system/config/wlan0`` – Wi-Fi settings (SSID, passphrase,
  channel, etc.)
- ``GET/POST /api/v1/system/config/wwan0`` – cellular modem settings (APN, PIN,
  roaming, etc.)
- ``GET/POST /api/v1/system/config/device`` – hostname, device name, location
- ``GET/POST /api/v1/system/config/communication-leds`` – LED target
  assignments (red, green)
- ``GET/POST /api/v1/system/config/debugging`` – debug/trace logging, filter
  rules, process metrics
- ``GET/POST /api/v1/system/config/services`` – SSH, VictoriaMetrics, Docker,
  memory monitor, MQTT broker, OpenVPN
- ``GET/POST /api/v1/system/config/tls`` – CA certificate, device certificate,
  private key (base64-encoded PEM)
- ``GET/POST /api/v1/system/config/smtp`` – SMTP server, port, encryption,
  authentication, sender, password

All configuration object endpoints require **SystemAdministrator**
authentication.

Example – reading the device configuration:

.. code-block:: http

  GET /api/v1/system/config/device HTTP/1.1
  Authorization: Bearer <accessToken>

Example response:

.. code-block:: json

  {
    "name": { "data": "Central Gateway" },
    "uuid": { "data": "abc123def456" },
    "state": { "data": true },
    "location": { "data": "Building 1, Room 234" }
  }

Example – writing the device configuration:

.. code-block:: http

  POST /api/v1/system/config/device HTTP/1.1
  Authorization: Bearer <accessToken>
  Content-Type: application/json

.. code-block:: json

  {
    "name": { "data": "Central Gateway (renamed)" },
    "location": { "data": "Building 2, Room 100" }
  }


System Update
-------------

.. _system-update:

The system update flow is a multi-step process: start an upload, send chunks,
then poll for installation state.

Endpoints
~~~~~~~~~

POST /api/v1/system/update/upload/start
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Begin a new update bundle upload.

- **Authentication:** SystemAdministrator
- **Request body** (JSON, max 1000 bytes):

  .. code-block:: json

    {
      "bytesToReceive": 536870912
    }

  - ``bytesToReceive`` (integer, required) – total size of the upload in bytes.
    Must be ≥ 1024 and ≤ 256 MiB.

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large, size too small, or size limit
    exceeded
  - ``500 Internal Server Error`` – failed to open output file

POST /api/v1/system/update/upload/chunk
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Upload a chunk of the update bundle.

- **Authentication:** SystemAdministrator
- **Request body** – raw binary data, max 4 MiB per chunk
- **Responses:**

  - ``200 OK`` – body is the current byte offset (string), e.g. ``"1048576"``
  - ``400 Bad Request`` – chunk too large, file not open, or size limit
    exceeded

GET /api/v1/system/update/install/state
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Poll the current installation progress.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "message": "Installing update bundle...",
      "progress": 45
    }

GET /api/v1/system/update/error
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve error data from the last (or current) update attempt.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – plain-text error description (may be empty)


Time-Series Database (VictoriaMetrics)
--------------------------------------

.. _timeseries:

Endpoints for querying, exporting, and managing time-series data stored in the
local VictoriaMetrics instance.

Endpoints
~~~~~~~~~

GET /api/v1/vmetrics/timeseries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve the list of available time-series (metric labels) for a given time
range.

- **Authentication:** SystemAdministrator
- **Query parameters:**

  - ``start`` (string, required) – start timestamp in milliseconds (≥ 13
    digits)
  - ``end`` (string, required) – end timestamp in milliseconds (≥ 13 digits)

- **Responses:**

  - ``200 OK`` – JSON array of series descriptors:

    .. code-block:: json

      [
        { "name": "eth0_rx[unit=Gateway][signal=Throughput]", "hash": "a1b2c3d4e5f6" },
        { "name": "cpu_usage[unit=Gateway]", "hash": "f6e5d4c3b2a1" }
      ]

  - ``202 Accepted`` – labels are still being fetched; retry later
  - ``400 Bad Request`` – missing or invalid ``start``/``end``

GET /api/v1/vmetrics/export/csv
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Export time-series data as a CSV file (chunked/streaming response).

- **Authentication:** SystemAdministrator
- **Query parameters:**

  - ``start`` (string, required) – start timestamp in milliseconds
  - ``end`` (string, required) – end timestamp in milliseconds
  - ``step`` (string, optional) – step size in seconds (default ``60``)
  - ``rollup`` (string, optional) – comma-separated rollup functions:
    ``min``, ``max``, ``avg``, ``sum``, ``cnt``
  - ``series`` (string, required) – comma-separated list of series hashes
  - ``decimalSeparator`` (string, optional) – ``"."`` (default) or `","`
  - ``dateTimeFormat`` (string, optional) – ``timestamp`` (default),
    ``local``, ``utc``, ``localized``, ``iso``
  - ``dateTimeLocale`` (string, optional) – locale string, e.g. ``de_DE``
    (default ``C``)

- **Response:**

  - ``200 OK`` – chunked ``text/plain`` CSV with
    ``Content-Disposition: attachment; filename="SIINEOS-VictoriaMetrics-Export_<start>_<end>.csv"``
  - ``400 Bad Request`` – invalid parameters
  - ``401 Unauthorized`` – missing/invalid token

  CSV format: semicolon-separated. First column is the timestamp (formatted
  according to ``dateTimeFormat``), subsequent columns are one per requested
  series.

POST /api/v1/vmetrics/timeseries/delete
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Delete one or more time-series from the database.

- **Authentication:** SystemAdministrator
- **Request body** (JSON): array of series hash strings

  .. code-block:: json

    ["a1b2c3d4e5f6", "f6e5d4c3b2a1"]

- **Responses:**

  - ``200 OK`` – empty body (deletion successful)
  - ``400 Bad Request`` – empty or invalid body
  - ``404 Not Found`` – one or more series not found
  - ``500 Internal Server Error`` – HTTP error during deletion

POST /api/v1/vmetrics/reset
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Reset (wipe) the VictoriaMetrics database.

- **Authentication:** SystemAdministrator
- **Responses:**

  - ``200 OK`` – empty body
  - ``500 Internal Server Error`` – reset failed


I/O Management
--------------

.. _io-management:

The I/O subsystem manages **units** (physical or virtual I/O devices),
**signals** (measurable or controllable channels within a unit), and **ports**
(connection points on a unit). All endpoints require **SystemAdministrator**
authentication unless noted otherwise.

Units
~~~~~

GET /api/v1/io/units
^^^^^^^^^^^^^^^^^^^^^^^

List all I/O unit UUIDs.

- **Response ``200 OK``** – JSON array of UUID strings:

  .. code-block:: json

    ["907cc14b665a47f2963907f344f7bb73", "a1b2c3d4e5f60718293a4b5c6d7e8f90"]

GET /api/v1/io/units/{unitUuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve the announcement data for a single unit.

- **Path parameter:** ``unitUuid`` (string)
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "providerName": "Synthetic Signals",
      "providerId": "synthetic_signals",
      "providerType": 4,
      "providerConnectivity": 0,
      "uuid": "907cc14b665a47f2963907f344f7bb73",
      "address": "",
      "name": "Synthetic signals",
      "location": "",
      "enabled": true,
      "connected": true,
      "error": "",
      "icon": "icon.svg",
      "busy": false,
      "signalsBrowsable": false,
      "signalsResettable": false,
      "signalAddressesContainPinAssignment": false,
      "signals": ["sig-uuid-1", "sig-uuid-2"],
      "upstreamUnit": "",
      "upstreamPort": "",
      "ports": []
    }

- **Response ``404 Not Found``** – unit not found

POST /api/v1/io/units
^^^^^^^^^^^^^^^^^^^^^^^^

Create a new I/O unit instance for the given provider.

- **Request body** – plain-text provider identifier (e.g. ``"modbus_client"``)
- **Responses:**

  - ``200 OK`` – body is the new unit's UUID
  - ``400 Bad Request`` – provider not found

DELETE /api/v1/io/units/{unitUuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove a unit and its configuration.

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – unit not found

Unit Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Unit settings are stored as a **configuration object** with the ``"data"``
sub-key pattern.

GET /api/v1/io/units/{unitUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read the full unit configuration.

- **Response ``200 OK``** – JSON with ``"data"`` sub-keys:

  .. code-block:: json

    {
      "general": {
        "name": { "data": "My Modbus Unit" },
        "uuid": { "data": "a1b2c3d4..." },
        "state": { "data": true },
        "providerId": { "data": "modbus_client" },
        "upstreamUnit": { "data": "" },
        "upstreamPort": { "data": "" },
        "signals": { "data": [ ... ] }
      },
      "data": {
        "customIcon": { "data": "" }
      }
    }

PUT /api/v1/io/units/{unitUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Replace the full unit configuration.

- **Request body** (JSON, max 4 MiB) – same structure as the GET response.
  The ``providerId`` must match the unit's actual provider; the ``uuid`` is
  overwritten with the path parameter.
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid JSON or body too large
  - ``404 Not Found`` – unit not found
  - ``409 Conflict`` – provider ID mismatch

GET /api/v1/io/units/{unitUuid}/config/{configObjectId}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read a specific configuration sub-object (e.g. ``"general"``, ``"data"``, or a
signal's ``"signal"`` object).

- **Path parameters:** ``unitUuid``, ``configObjectId``
- **Response ``200 OK``** – JSON with ``"data"`` sub-keys for that sub-object
- **Response ``404 Not Found``** – unit or config object not found

PUT /api/v1/io/units/{unitUuid}/config/{configObjectId}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write a specific configuration sub-object.

- **Request body** (JSON, max 4 MiB)
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid JSON
  - ``404 Not Found`` – unit or config object not found

Unit Name, Icon, and Upstream
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

PUT /api/v1/io/units/{unitUuid}/name
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Rename a unit.

- **Request body** – plain-text name, max 256 bytes
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large
  - ``404 Not Found`` – unit not found

GET /api/v1/io/units/{unitUuid}/icon
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve the unit's custom icon (binary).

- **Authentication:** none
- **Query parameter:** ``<hash>`` (string) – hash of the icon data (for
  cache-busting)
- **Response ``200 OK``** – binary image data
- **Response ``404 Not Found``** – unit not found

PUT /api/v1/io/units/{unitUuid}/icon
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Upload a custom icon for the unit.

- **Request body** – binary image data, max 128 KiB
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large
  - ``404 Not Found`` – unit not found

DELETE /api/v1/io/units/{unitUuid}/icon
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove the custom icon (reverts to the provider default).

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – unit not found

PUT /api/v1/io/units/{unitUuid}/upstream
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Set the upstream unit and port for daisy-chained units.

- **Request body** (JSON, max 256 bytes):

  .. code-block:: json

    {
      "unit": "upstream-unit-uuid",
      "port": "port-uuid"
    }

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large or invalid JSON
  - ``404 Not Found`` – unit not found

Unit Control
^^^^^^^^^^^^^^^^^^^^

POST /api/v1/io/units/{unitUuid}/control/{action}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Send a provider-specific control command to a unit.

- **Path parameters:** ``unitUuid``, ``action`` (provider-defined string)
- **Request body** – provider-specific payload, max 4 MiB
- **Responses:**

  - ``200 OK`` – provider-specific response
  - ``400 Bad Request`` – body too large
  - ``404 Not Found`` – unit not found
  - ``501 Not Implemented`` – unit does not support control

Ports
~~~~~

GET /api/v1/io/units/{unitUuid}/ports/{portUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read a port's configuration.

- **Response ``200 OK``** – JSON with ``"data"`` sub-keys
- **Response ``404 Not Found``** – unit or port not found

POST /api/v1/io/units/{unitUuid}/ports/{portUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write a port's configuration.

- **Request body** (JSON) – same structure as GET response
- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – unit or port not found

Signals
~~~~~~~

GET /api/v1/io/units/{unitUuid}/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

List signal UUIDs for a unit.

- **Response ``200 OK``** – JSON array of UUID strings

GET /api/v1/io/units/{unitUuid}/signals/{signalUuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieve a single signal's announcement data.

- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "uuid": "sig-uuid-1",
      "type": "Temperature",
      "identifier": "temp_1",
      "address": "0",
      "name": "Temperature Sensor 1",
      "group": "Sensors",
      "dataSeriesSet": "",
      "readable": true,
      "writable": false,
      "enabled": true,
      "dataType": 6,
      "unit": "°C",
      "decimals": 1,
      "minValue": -40,
      "maxValue": 125,
      "color": "#00dbc9",
      "visualization": "gauge"
    }

POST /api/v1/io/units/{unitUuid}/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add a single new signal to a unit.

- **Response ``200 OK``** – body is the new signal's UUID
- **Response ``404 Not Found``** – unit not found

PUT /api/v1/io/units/{unitUuid}/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add multiple signals to a unit in one request.

- **Request body** (JSON): array of signal property objects

  .. code-block:: json

    [
      { "name": { "data": "Signal A" }, "address": { "data": "0" } },
      { "name": { "data": "Signal B" }, "address": { "data": "1" } }
    ]

- **Response ``200 OK``** – JSON array of new signal UUIDs
- **Response ``400 Bad Request``** – invalid JSON
- **Response ``404 Not Found``** – unit not found

DELETE /api/v1/io/units/{unitUuid}/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove multiple signals from a unit.

- **Query parameter:** ``uuids`` (string, required) – comma-separated signal
  UUIDs

  Example: ``?uuids=sig-1,sig-2,sig-3``

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – missing ``uuids`` parameter
  - ``404 Not Found`` – unit not found
  - ``500 Internal Server Error`` – removal failed

POST /api/v1/io/units/{unitUuid}/signals/{signalUuid}/duplicate
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Duplicate an existing signal with a new name.

- **Request body** (JSON):

  .. code-block:: json

    {
      "name": "Temperature Sensor 1 (copy)"
    }

- **Responses:**

  - ``200 OK`` – body is the new signal's UUID
  - ``400 Bad Request`` – missing ``name`` field
  - ``404 Not Found`` – unit or source signal not found
  - ``500 Internal Server Error`` – duplication failed

Signal Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Signal settings use the **configuration object** pattern with ``"data"``
sub-keys. See :ref:`signal-processing-chains` for the full key reference.

GET /api/v1/io/units/{unitUuid}/signals/{signalUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read a signal's configuration.

- **Response ``200 OK``** – JSON with ``"data"`` sub-keys (see
  :ref:`signal-processing-chains`)

PUT /api/v1/io/units/{unitUuid}/signals/{signalUuid}/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write a signal's configuration.

- **Request body** (JSON) – same structure as GET response
- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – unit or signal not found

PUT /api/v1/io/units/{unitUuid}/signals/config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write configuration for multiple signals in one request.

- **Query parameter:** ``uuids`` (string, required) – comma-separated signal
  UUIDs
- **Request body** (JSON) – configuration object (applied to all listed
  signals)
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid JSON
  - ``404 Not Found`` – unit or any signal not found

PUT /api/v1/io/units/{unitUuid}/signals/clear
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove all signals from a unit.

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – unit not found
  - ``500 Internal Server Error`` – clear failed

PUT /api/v1/io/units/{unitUuid}/signals/reset
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Reset specific signals (e.g. zero counters).

- **Request body** (JSON): array of signal UUIDs

  .. code-block:: json

    ["sig-uuid-1", "sig-uuid-2"]

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid JSON
  - ``404 Not Found`` – unit not found

POST /api/v1/io/units/{unitUuid}/signals/{signalUuid}/control/{action}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Send a signal-specific control command.

- **Path parameters:** ``unitUuid``, ``signalUuid``, ``action``
- **Request body** – provider-specific payload, max 4 MiB
- **Responses:**

  - ``200 OK`` – provider-specific response
  - ``400 Bad Request`` – missing action or body too large
  - ``404 Not Found`` – unit or signal not found
  - ``501 Not Implemented`` – signal does not support control

I/O Connections
^^^^^^^^^^^^^^^^^^^^

I/O connections define signal processing chains that route a source signal's
output to a destination signal's input. The collection is a **plain JSON array**
(no ``"data"`` wrapper).

Connection Object Schema
~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required) – unique identifier
- ``source`` (object, required):

  - ``unit`` (string) – source unit UUID
  - ``signal`` (string) – source signal UUID

- ``destination`` (object, required):

  - ``unit`` (string) – destination unit UUID
  - ``signal`` (string) – destination signal UUID

- ``signalProcessings`` (array, optional) – array of processing chain
  configuration objects (see :ref:`signal-processing-chains`)
- ``disabled`` (boolean, optional) – whether the connection is disabled

Endpoints:

- ``GET /api/v1/io/connections`` – list all connections
- ``GET /api/v1/io/connections/{uuid}`` – get one connection
- ``POST /api/v1/io/connections`` – create a connection
- ``PUT /api/v1/io/connections/{uuid}`` – modify a connection
- ``DELETE /api/v1/io/connections/{uuid}`` – delete a connection
- ``PUT /api/v1/io/connections/disable/{uuid}`` – enable/disable a connection

All require **SystemAdministrator** authentication.

Default Signal Processing Chain Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

GET /api/v1/io/signal-processing/config/default
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read the default (template) signal processing chain configuration. This is a
**read-only** configuration object.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON with ``"data"`` sub-keys (see
  :ref:`signal-processing-chains`)


Signal Processing Chains
------------------------

.. _signal-processing-chains:

Every I/O signal has an embedded **signal processing chain** that transforms
the raw measurement value through a sequence of optional stages before
producing the final output. The chain is configured via the signal's
configuration object and uses the ``"data"`` sub-key pattern.

Chain Stage Order
~~~~~~~~~~~~~~~~~

The processing stages are applied in the following fixed order:

1. Pre-processing (mathematical expression)
2. Linear scaling
3. Delta
4. Limit
5. Threshold comparison
6. Comparator
7. Edge detection
8. Time derivative
9. Aggregation
10. Post-processing (mathematical expression)

Configuration Keys Reference
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All keys below appear in the signal's configuration JSON. Each value is wrapped
in ``{ "data": <value> }``.

Pre-processing
^^^^^^^^^^^^^^^^

- ``preProcessingEnabled`` (boolean) – enable/disable the stage
- ``preProcessingExpression`` (string) – mathematical expression using
  variable ``x`` for the input value. Example: ``"1.25 + (x / 2)"``

Linear Scaling
^^^^^^^^^^^^^^^^

- ``linearScalingEnabled`` (boolean)
- ``linearScalingX1`` (float) – minimum input value
- ``linearScalingY1`` (float) – corresponding minimum output value
- ``linearScalingX2`` (float) – maximum input value
- ``linearScalingY2`` (float) – corresponding maximum output value

Delta
^^^^^

- ``deltaEnabled`` (boolean)
- ``deltaMode`` (unsigned integer):

  - ``0`` – absolute difference to previous value
  - ``1`` – relative change from previous value
  - ``2`` – relative change in percent
  - ``3`` – sign difference from previous value

Limit
^^^^^

- ``limitEnabled`` (boolean)
- ``limitDownwards`` (boolean) – apply lower limit
- ``limitMinimumValue`` (float) – lower bound
- ``limitUpwards`` (boolean) – apply upper limit
- ``limitMaximumValue`` (float) – upper bound

Threshold Comparison
^^^^^^^^^^^^^^^^^^^^^^^^

- ``thresholdComparisonEnabled`` (boolean)
- ``thresholdComparisonMode`` (unsigned integer):

  - ``0`` – signal is above threshold
  - ``1`` – signal is above or equal
  - ``2`` – signal is below threshold
  - ``3`` – signal is below or equal

- ``thresholdComparisonValue`` (unsigned integer) – threshold value

Comparator
^^^^^^^^^^

- ``comparatorEnabled`` (boolean)
- ``comparatorLowerThreshold`` (float)
- ``comparatorUpperThreshold`` (float)
- ``comparatorOutputValueOff`` (float) – output when below lower threshold
- ``comparatorOutputValueOn`` (float) – output when above upper threshold
- ``comparatorTimerIntervalOff`` (unsigned integer, ms) – required duration
  below lower threshold
- ``comparatorTimerIntervalOn`` (unsigned integer, ms) – required duration
  above upper threshold

Edge Detection
^^^^^^^^^^^^^^^^

- ``edgeDetectionEnabled`` (boolean)
- ``edgeDetectionCountRising`` (boolean) – count rising edges
- ``edgeDetectionCountFalling`` (boolean) – count falling edges

Time Derivative
^^^^^^^^^^^^^^^^^

- ``timeDerivativeEnabled`` (boolean)
- ``timeDerivativeTimeBase`` (unsigned integer):

  - ``0`` – per second (Δx/s)
  - ``1`` – per minute (Δx/min)
  - ``2`` – per hour (Δx/h)
  - ``3`` – per day (Δx/d)

Aggregation
^^^^^^^^^^^^^

- ``aggregationEnabled`` (boolean)
- ``aggregationType`` (unsigned integer):

  - ``0`` – minimum value
  - ``1`` – maximum value
  - ``2`` – average value
  - ``3`` – oldest value
  - ``4`` – sum of all values
  - ``5`` – median

- ``aggregationInterval`` (unsigned integer, seconds) – window size
- ``aggregationUpdateMode`` (unsigned integer):

  - ``0`` – continuously (update on every sample)
  - ``1`` – periodically (update at end of interval)

Post-processing
^^^^^^^^^^^^^^^^^

- ``postProcessingEnabled`` (boolean)
- ``postProcessingExpression`` (string) – mathematical expression using
  variable ``x``. Example: ``"x * 1.05"``

Signal Metadata Keys
^^^^^^^^^^^^^^^^^^^^^^^^

In addition to the processing chain keys, the signal configuration includes:

- ``name`` (string) – display name
- ``uuid`` (string, read-only) – signal UUID
- ``enabled`` (boolean) – enable/disable the signal
- ``vmetricsPushEnabled`` (boolean) – record values to VictoriaMetrics
- ``vmetricsPushInterval`` (unsigned integer, seconds) – recording interval
- ``useCustomIdentifier`` (boolean)
- ``customIdentifier`` (string) – custom identifier (overrides auto-generated)
- ``group`` (string) – grouping label
- ``dataSeriesSet`` (string) – data series set label
- ``siPrefix`` (unsigned integer) – SI prefix (0 = none, 3 = k, 9 = µ, etc.)
- ``measurementUnit`` (string) – unit string (e.g. ``"DegreeCelsius"``)
- ``decimals`` (unsigned integer) – number of decimal places (0–5)
- ``customDataType`` (unsigned integer) – override data type (0 = keep
  original)
- ``visualizationType`` (string) – ``"none"``, ``"gauge"``, ``"led"``,
  ``"counter"``
- ``color`` (string) – hex colour or named colour
- ``minimumValue`` (double) – display minimum
- ``maximumValue`` (double) – display maximum

Example – Full Signal Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: json

  {
    "name": { "data": "Motor Temperature" },
    "uuid": { "data": "a1b2c3d4e5f6" },
    "enabled": { "data": true },
    "vmetricsPushEnabled": { "data": true },
    "vmetricsPushInterval": { "data": 60 },
    "useCustomIdentifier": { "data": false },
    "customIdentifier": { "data": "" },
    "group": { "data": "Motor" },
    "dataSeriesSet": { "data": "" },
    "siPrefix": { "data": 0 },
    "measurementUnit": { "data": "DegreeCelsius" },
    "decimals": { "data": 1 },
    "customDataType": { "data": 0 },
    "visualizationType": { "data": "gauge" },
    "color": { "data": "#00dbc9" },
    "minimumValue": { "data": -40 },
    "maximumValue": { "data": 150 },
    "preProcessingEnabled": { "data": false },
    "preProcessingExpression": { "data": "x" },
    "linearScalingEnabled": { "data": true },
    "linearScalingX1": { "data": 0 },
    "linearScalingY1": { "data": -40 },
    "linearScalingX2": { "data": 255 },
    "linearScalingY2": { "data": 150 },
    "deltaEnabled": { "data": false },
    "deltaMode": { "data": 0 },
    "limitEnabled": { "data": true },
    "limitDownwards": { "data": true },
    "limitMinimumValue": { "data": -40 },
    "limitUpwards": { "data": true },
    "limitMaximumValue": { "data": 150 },
    "thresholdComparisonEnabled": { "data": false },
    "thresholdComparisonMode": { "data": 0 },
    "thresholdComparisonValue": { "data": 0 },
    "comparatorEnabled": { "data": false },
    "comparatorLowerThreshold": { "data": 0 },
    "comparatorUpperThreshold": { "data": 0 },
    "comparatorOutputValueOff": { "data": 0 },
    "comparatorOutputValueOn": { "data": 1 },
    "comparatorTimerIntervalOff": { "data": 0 },
    "comparatorTimerIntervalOn": { "data": 0 },
    "edgeDetectionEnabled": { "data": false },
    "edgeDetectionCountRising": { "data": false },
    "edgeDetectionCountFalling": { "data": false },
    "timeDerivativeEnabled": { "data": false },
    "timeDerivativeTimeBase": { "data": 0 },
    "aggregationEnabled": { "data": false },
    "aggregationType": { "data": 2 },
    "aggregationInterval": { "data": 60 },
    "aggregationUpdateMode": { "data": 0 },
    "postProcessingEnabled": { "data": false },
    "postProcessingExpression": { "data": "x" }
  }


Synthetic Signals
-----------------

.. _synthetic-signals:

Synthetic signals are virtual I/O signals computed from other signals using
arithmetic or logical expressions. The collection is a **plain JSON array**.

Synthetic Signal Object Schema
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required) – unique identifier
- ``name`` (string, required) – display name
- ``enabled`` (boolean, optional) – defaults to ``true``
- ``source1`` (object, required):

  - ``unit`` (string) – source unit UUID
  - ``signal`` (string) – source signal UUID

- ``source2`` (object, required):

  - ``unit`` (string) – second source unit UUID
  - ``signal`` (string) – second source signal UUID

- ``calculation`` (string, required) – the calculation expression. Supported
  values:

  - ``"+"``, ``"-"``, ``"*"``, ``"/"`` – arithmetic on A and B
  - ``"&&"``, ``"||"`` – logical AND / OR
  - ``".."`` – latch (A sets to 1, B resets to 0)
  - ``"+."`` – counter with reset (A = count, B = reset trigger)
  - ``"++"`` – counter without reset (A = count)
  - Any other string – evaluated as a mathematical expression with variables
    ``A``, ``B``, ``P`` (previous value) and access to ``Math.*`` functions

Endpoints
~~~~~~~~~

GET /api/v1/io/synthetic/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

List all synthetic signals.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON array

GET /api/v1/io/synthetic/signals/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Get a single synthetic signal.

- **Response ``200 OK``** – single object
- **Response ``404 Not Found``** – not found

POST /api/v1/io/synthetic/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a synthetic signal.

- **Request body** (JSON):

  .. code-block:: json

    {
      "name": "Flow Difference",
      "source1": { "unit": "unit-uuid-1", "signal": "sig-uuid-1" },
      "source2": { "unit": "unit-uuid-2", "signal": "sig-uuid-2" },
      "calculation": "-"
    }

- **Responses:**

  - ``200 OK`` – body is the new UUID
  - ``400 Bad Request`` – invalid body

PUT /api/v1/io/synthetic/signals/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Modify a synthetic signal.

- **Request body** (JSON) – same schema as create
- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – not found

PUT /api/v1/io/synthetic/signals
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Replace all synthetic signals at once.

- **Request body** (JSON) – array of synthetic signal objects
- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid body

DELETE /api/v1/io/synthetic/signals/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Delete a synthetic signal.

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – not found

PUT /api/v1/io/synthetic/signals/disable/{uuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Enable/disable a synthetic signal.

- **Request body:** ``{ "disabled": true }``

POST /api/v1/io/synthetic/signals/{sourceUuid}/clone/{destUuid}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Clone the calculation settings from one synthetic signal to another.

- **Path parameters:** ``sourceUuid``, ``destUuid``
- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – either signal not found

POST /api/v1/io/synthetic/signals/{uuid}/name/propagate
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Propagate the name from the registry entry to the live I/O signal.

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – signal not found

POST /api/v1/io/synthetic/signals/{uuid}/name/update
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Update the registry name to match the live I/O signal name.

- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – signal not found
  - ``500 Internal Server Error`` – registry update failed


Modbus Server Endpoint
----------------------

.. _modbus-server:

The built-in Modbus server exposes TCP and RTU interfaces with configurable
registers.

Endpoints
~~~~~~~~~

GET/POST /api/v1/io/endpoints/modbus/tcp
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read/write Modbus TCP server settings.

- **Authentication:** SystemAdministrator
- **Configuration keys** (``"data"`` sub-key pattern):

  - ``enabled`` (boolean) – enable/disable the TCP server
  - ``networkAddress`` (string) – listen address (default ``"0.0.0.0"``)
  - ``networkPort`` (unsigned integer) – listen port (default ``502``)

GET/POST /api/v1/io/endpoints/modbus/rtu
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read/write Modbus RTU slave settings.

- **Authentication:** SystemAdministrator
- **Configuration keys:**

  - ``enabled`` (boolean)
  - ``rtuBusInterface`` (unsigned integer):

    - ``0`` – serial port
    - ``1`` – builtin RS485 interface
    - ``2`` – backplane bus

  - ``rtuSerialPortName`` (string) – e.g. ``"ttyUSB0"``
  - ``rtuBaudRate`` (unsigned integer) – e.g. ``115200``
  - ``rtuDataBits`` (unsigned integer) – 5, 6, 7, or 8
  - ``rtuParity`` (unsigned integer):

    - ``0`` – no parity
    - ``2`` – even
    - ``3`` – odd
    - ``4`` – space
    - ``5`` – mark

  - ``rtuStopBits`` (unsigned integer):

    - ``1`` – 1 stop bit
    - ``2`` – 2 stop bits
    - ``3`` – 1.5 stop bits

  - ``modbusId`` (unsigned integer) – Modbus slave address (1–254)

GET/POST/PUT/DELETE /api/v1/io/endpoints/modbus/registers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Manage Modbus register mappings. The collection is a **plain JSON array**.

Register Object Schema
~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required) – unique identifier
- ``type`` (integer) – Modbus register type
- ``address`` (integer) – register address
- ``dataType`` (integer, optional) – data type:

  - ``0`` – unsigned small integer (1 register)
  - ``1`` – signed small integer (1 register)
  - ``2`` – unsigned integer (2 registers)
  - ``3`` – signed integer (2 registers)
  - ``4`` – float (2 registers)
  - ``5`` – unsigned big integer (4 registers)
  - ``6`` – signed big integer (4 registers)
  - ``7`` – double (4 registers)

- ``ioMode`` (integer) – ``0`` = read from signal, ``1`` = write to signal
- ``signal`` (object):

  - ``unit`` (string) – target unit UUID
  - ``signal`` (string) – target signal UUID

Example register:

.. code-block:: json

  {
    "uuid": "reg-uuid-1",
    "type": 3,
    "address": 0,
    "dataType": 4,
    "ioMode": 0,
    "signal": { "unit": "unit-uuid-1", "signal": "sig-uuid-1" }
  }


Firewall
--------

.. _firewall:

Firewall rules are managed as four separate collections (all **plain JSON
arrays** with move-up/move-down support) plus internet connection sharing
settings.

Rule Object Schema (common fields)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required)
- ``enabled`` (boolean, optional, default ``true``)
- ``networkProtocol`` (string) – protocol identifier
- ``inputInterface`` / ``outputInterface`` (string) – interface name
- ``sourceAddress`` / ``destinationAddress`` (string) – IP or CIDR
- ``destinationPorts`` (string) – space-separated port list
- ``action`` (string) – rule action (accept/drop)

Endpoints
~~~~~~~~~

Incoming Rules
^^^^^^^^^^^^^^^^

- ``GET /api/v1/firewall/rules/incoming`` – list
- ``GET /api/v1/firewall/rules/incoming/{uuid}`` – get one
- ``POST /api/v1/firewall/rules/incoming`` – create
- ``PUT /api/v1/firewall/rules/incoming/{uuid}`` – modify
- ``DELETE /api/v1/firewall/rules/incoming/{uuid}`` – delete
- ``PUT /api/v1/firewall/rules/incoming/{uuid}/up`` – move up
- ``PUT /api/v1/firewall/rules/incoming/{uuid}/down`` – move down

Outgoing Rules
^^^^^^^^^^^^^^^^

- ``GET /api/v1/firewall/rules/outgoing`` – list
- ``GET /api/v1/firewall/rules/outgoing/{uuid}`` – get one
- ``POST /api/v1/firewall/rules/outgoing`` – create
- ``PUT /api/v1/firewall/rules/outgoing/{uuid}`` – modify
- ``DELETE /api/v1/firewall/rules/outgoing/{uuid}`` – delete
- ``PUT /api/v1/firewall/rules/outgoing/{uuid}/up`` – move up
- ``PUT /api/v1/firewall/rules/outgoing/{uuid}/down`` – move down

IP Forwarding Rules
^^^^^^^^^^^^^^^^^^^^^

- ``GET /api/v1/firewall/rules/ipforwarding`` – list
- ``GET /api/v1/firewall/rules/ipforwarding/{uuid}`` – get one
- ``POST /api/v1/firewall/rules/ipforwarding`` – create
- ``PUT /api/v1/firewall/rules/ipforwarding/{uuid}`` – modify
- ``DELETE /api/v1/firewall/rules/ipforwarding/{uuid}`` – delete
- ``PUT /api/v1/firewall/rules/ipforwarding/{uuid}/up`` – move up
- ``PUT /api/v1/firewall/rules/ipforwarding/{uuid}/down`` – move down

Additional fields for IP forwarding rules:

- ``inputInterface`` (string)
- ``outputInterface`` (string)
- ``sourceAddress`` (string)
- ``destinationAddress`` (string)

Port Forwarding Rules
^^^^^^^^^^^^^^^^^^^^^^^

- ``GET /api/v1/firewall/rules/portforwarding`` – list
- ``GET /api/v1/firewall/rules/portforwarding/{uuid}`` – get one
- ``POST /api/v1/firewall/rules/portforwarding`` – create
- ``PUT /api/v1/firewall/rules/portforwarding/{uuid}`` – modify
- ``DELETE /api/v1/firewall/rules/portforwarding/{uuid}`` – delete
- ``PUT /api/v1/firewall/rules/portforwarding/{uuid}/up`` – move up
- ``PUT /api/v1/firewall/rules/portforwarding/{uuid}/down`` – move down

Additional fields for port forwarding rules:

- ``localPort`` (string) – local port to forward to
- ``destinationPort`` (string) – destination port
- ``destinationAddress`` (string) – target IP
- ``masquerade`` (boolean, optional) – enable masquerade

Internet Connection Sharing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

GET /api/v1/firewall/internet/sharing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Read internet connection sharing settings.

- **Authentication:** SystemAdministrator
- **Response ``200 OK``** – JSON:

  .. code-block:: json

    {
      "enabled": true,
      "inputInterfaces": ["eth0", "wlan0"],
      "outputInterface": "wwan0"
    }

POST /api/v1/firewall/internet/sharing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Write internet connection sharing settings.

- **Authentication:** SystemAdministrator
- **Request body** (JSON, max 100000 bytes):

  .. code-block:: json

    {
      "enabled": true,
      "inputInterfaces": ["eth0", "wlan0"],
      "outputInterface": "wwan0"
    }

  - ``enabled`` (boolean, required)
  - ``inputInterfaces`` (array of strings, required) – interfaces to share
    from
  - ``outputInterface`` (string, required) – interface to share through.
    Must be one of: ``eth0``, ``eth1``, ``wlan0``, ``wwan0``, ``tun0``,
    ``docker0``

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – invalid body or unknown interface


Alerting
--------

.. _alerting:

The alerting subsystem comprises three collections: **signals** (conditions to
monitor), **destinations** (notification targets), and **rules** (bindings
between signals and destinations). All are **plain JSON arrays** with
disable support.

Alert Signal Object Schema
~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required)
- ``name`` (string, required)
- ``disabled`` (boolean, optional)
- ``evalMode`` (string, required) – one of:

  - ``"binary"``
  - ``"thresholds"``
  - ``"counter"``
  - ``"cycles"``

- ``evalParams`` (object, required):

  - For ``binary``: ``lowActive`` (boolean)
  - For ``thresholds``: ``comparison`` (string: ``"above"``,
    ``"aboveOrEqual"``, ``"below"``, ``"belowOrEqual"``, ``"equal"``,
    ``"insideRange"``, ``"outsideRange"``), ``lowerThreshold`` (number),
    ``upperThreshold`` (number, for range comparisons)
  - For ``counter``: ``counterStepSize`` (number)
  - For ``cycles``: ``pulseThreshold`` (number)

- ``source`` (object, required):

  - ``unit`` (string) – source I/O unit UUID
  - ``signal`` (string) – source I/O signal UUID

- ``cycleTime`` (number, optional) – timeout in seconds for counter/cycles
  modes
- ``stateTransition`` (object, optional):

  - ``alarmDelayEnabled`` (boolean)
  - ``alarmDelay`` (number, seconds)
  - ``okDelayEnabled`` (boolean)
  - ``okDelay`` (number, seconds)

- ``severity`` (string, optional) – ``"none"``, ``"low"``, ``"medium"``,
  ``"high"``
- ``category`` (string, optional) – free-text category label

Alert Destination Object Schema
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required)
- ``name`` (string, required)
- ``disabled`` (boolean, optional)
- ``type`` (string, required) – one of: ``"email"``, ``"sms"``, ``"webhook"``,
  ``"app"``, ``"iosignal"``, ``"mqtt"``, ``"vmetrics"``
- Type-specific sub-objects:

  - ``email``: ``recipients`` (string, comma-separated), ``subject`` (string),
    ``body`` (string)
  - ``sms``: ``recipients`` (string), ``text`` (string)
  - ``webhook``: ``url`` (string), ``body`` (string), ``method`` (string:
    ``"GET"``, ``"POST"``, ``"PUT"``, ``"DELETE"``)
  - ``app``: ``appId`` (string)
  - ``iosignal``: ``unit`` (string), ``signal`` (string)
  - ``mqtt``: ``brokerAddress`` (string), ``brokerPort`` (integer),
    ``topic`` (string), ``data`` (string)
  - ``vmetrics``: ``metricName`` (string)

  Placeholder tokens available in text fields: ``{{NAME}}``, ``{{STATE}}``,
  ``{{VALUE}}``, ``{{SEVERITY}}``, ``{{CATEGORY}}``, ``{{CLASSIFICATION}}``,
  ``{{ID}}``, ``{{SOURCENAME}}``, ``{{SOURCEVALUE}}``, ``{{SOURCELOCATION}}``,
  ``{{HOSTNAME}}``, ``{{DEVICENAME}}``, ``{{DEVICELOCATION}}``

Alert Rule Object Schema
~~~~~~~~~~~~~~~~~~~~~~~~

- ``uuid`` (string, required)
- ``name`` (string, required)
- ``disabled`` (boolean, optional)
- ``alertSignals`` (array, required) – array of ``{ "uuid": "<signal-uuid>" }``
  objects, or ``[{ "uuid": "*" }]`` to match all signals
- ``destinations`` (array, required) – array of ``{ "uuid":
  "<destination-uuid>" }`` objects
- ``triggers`` (object, required):

  - ``ok`` (boolean) – trigger on OK state
  - ``alarm`` (boolean) – trigger on alarm state

- ``severityLevels`` (object, required):

  - ``all`` (boolean) – match all severities
  - ``none`` (boolean) – include "no severity"
  - ``low`` (boolean)
  - ``medium`` (boolean)
  - ``high`` (boolean)

- ``repetition`` (integer, optional) – re-notification interval in minutes
  (0 or omitted = no repetition)

Endpoints
~~~~~~~~~

Alert Signals
^^^^^^^^^^^^^^^

- ``GET /api/v1/alerting/signals`` – list
- ``GET /api/v1/alerting/signals/{uuid}`` – get one
- ``POST /api/v1/alerting/signals`` – create
- ``PUT /api/v1/alerting/signals/{uuid}`` – modify
- ``DELETE /api/v1/alerting/signals/{uuid}`` – delete
- ``PUT /api/v1/alerting/signals/disable/{uuid}`` – enable/disable

Alert Destinations
^^^^^^^^^^^^^^^^^^^^

- ``GET /api/v1/alerting/destinations`` – list
- ``GET /api/v1/alerting/destinations/{uuid}`` – get one
- ``POST /api/v1/alerting/destinations`` – create
- ``PUT /api/v1/alerting/destinations/{uuid}`` – modify
- ``DELETE /api/v1/alerting/destinations/{uuid}`` – delete
- ``PUT /api/v1/alerting/destinations/disable/{uuid}`` – enable/disable

Alert Rules
^^^^^^^^^^^^^

- ``GET /api/v1/alerting/rules`` – list
- ``GET /api/v1/alerting/rules/{uuid}`` – get one
- ``POST /api/v1/alerting/rules`` – create
- ``PUT /api/v1/alerting/rules/{uuid}`` – modify
- ``DELETE /api/v1/alerting/rules/{uuid}`` – delete
- ``PUT /api/v1/alerting/rules/disable/{uuid}`` – enable/disable

All alerting endpoints require **SystemAdministrator** authentication.


Licensing
---------

.. _licensing:

License management for the SIINEOS platform and third-party applications.

Endpoints
~~~~~~~~~

GET /api/v1/licensing/licenses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

List all installed licenses.

- **Authentication:** none
- **Response ``200 OK``** – JSON array:

  .. code-block:: json

    [
      {
        "id": "lic-uuid-1",
        "productId": "de.inhub.siineos",
        "productName": "SIINEOS Hub",
        "productVersion": "1.2.3",
        "productSize": 512,
        "valid": true,
        "validFrom": "2024-01-01",
        "validUntil": "2027-01-01",
        "licensee": "ACME Corp"
      }
    ]

POST /api/v1/licensing/licenses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Import a license by uploading a license certificate file.

- **Authentication:** SystemAdministrator
- **Request body** – raw binary license certificate file (max 100000 bytes).
  The server parses the certificate and extracts license metadata internally.
  **No JSON is sent by the client.**
- **Responses:**

  - ``200 OK`` – body is JSON with the imported license metadata:

    .. code-block:: json

      {
        "id": "lic-uuid-1",
        "productId": "de.inhub.siineos",
        "productName": "SIINEOS Hub",
        "productVersion": "1.2.3",
        "productSize": 512,
        "valid": true,
        "validFrom": "2024-01-01",
        "validUntil": "2027-01-01",
        "licensee": "ACME Corp"
      }

  - ``400 Bad Request`` – body too large
  - ``409 Conflict`` – a license with the same ID is already installed
  - ``415 Unsupported Media Type`` – the certificate is not valid
  - ``422 Unprocessable Entity`` – the certificate is valid but the license
    is not accepted (e.g. expired)

DELETE /api/v1/licensing/licenses/{licenseId}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove a license.

- **Authentication:** SystemAdministrator
- **Path parameter:** ``licenseId`` (string)
- **Responses:**

  - ``200 OK`` – empty body
  - ``404 Not Found`` – license not found
  - ``500 Internal Server Error`` – removal failed


Apps
----

.. _apps:

Application control endpoints for managing installed apps on the hub.

Endpoints
~~~~~~~~~

PUT /api/v1/apps/{appId}/control/{controlProperty}
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Set a control property on an app.

- **Authentication:** SystemAdministrator
- **Path parameters:**

  - ``appId`` (string) – application identifier
  - ``controlProperty`` (string) – one of: ``"enabled"``, ``"debug"``,
    ``"trace"``

- **Request body** – JSON boolean (max 10 bytes)

  Example: ``true``

- **Responses:**

  - ``200 OK`` – empty body
  - ``400 Bad Request`` – body too large or invalid JSON
  - ``404 Not Found`` – unknown control property


Ping / Health
-------------

.. _ping:

GET /api/v1/ping
~~~~~~~~~~~~~~~~

Health-check endpoint.

- **Authentication:** none
- **Response ``200 OK``** – empty body


Error Handling
--------------

The API uses standard HTTP status codes. Error responses generally have an
empty body or a short plain-text message.

Status Code Reference
~~~~~~~~~~~~~~~~~~~~~

- ``400 Bad Request`` – malformed request body, missing required fields,
  body exceeds size limit, invalid parameter values, or a business-rule
  violation (e.g. deleting the protected admin account).

  Typical body: empty string or a short message such as
  ``"upload file size too small"`` or ``"body length limit exceeded"``.

- ``401 Unauthorized`` – missing, malformed, or expired access token; or
  invalid credentials during login.

  Typical body: ``"Authentication required"`` or empty.

- ``403 Forbidden`` – valid token but insufficient role (e.g. a non-admin
  user accessing an admin-only endpoint), or session ID mismatch.

  Typical body: ``"Admin privileges required"`` or
  ``"Invalid access token"``.

- ``404 Not Found`` – the requested resource (user, unit, signal, license,
  session, etc.) does not exist.

  Typical body: empty string.

- ``409 Conflict`` – a resource with the same unique key already exists
  (e.g. duplicate user login name, duplicate license ID, provider ID
  mismatch on unit config write).

  Typical body: empty string or the conflicting value.

- ``415 Unsupported Media Type`` – the uploaded file is not a valid license
  certificate.

- ``422 Unprocessable Entity`` – the request is syntactically correct but
  semantically invalid (e.g. license certificate is well-formed but the
  license is expired or not accepted).

- ``500 Internal Server Error`` – an unexpected server-side failure (e.g.
  file I/O error, external service failure, database write failure).

  Typical body: empty string.

- ``501 Not Implemented`` – the requested operation is not supported by the
  target unit or signal.

  Typical body: ``"Not implemented"``.

General Notes
~~~~~~~~~~~~~

- All JSON request bodies have a maximum size of **1 MiB** unless otherwise
  stated (e.g. 4 MiB for unit config writes, 128 KiB for icon uploads,
  256 MiB for update bundle chunks).
- UUIDs in the API are 32-character hexadecimal strings **without dashes**.
- Timestamps in query parameters for the time-series API are in
  **milliseconds** since epoch (13 digits).
- The ``"data"`` sub-key pattern applies exclusively to configuration-object
  endpoints. Collection endpoints (users, firewall rules, alerting, I/O
  connections, synthetic signals, Modbus registers) use plain JSON objects
  without the wrapper.

