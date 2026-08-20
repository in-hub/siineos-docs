SIINEOS HTTP API reference
==========================

Overview
--------

The SMAC-Server exposes a RESTful HTTP API on port **80**. All endpoints
are prefixed with ``/api/v1``. The API is served by an internal HTTP server
and provides access to system management, I/O control, alerting, licensing,
firewall configuration, time-series data, and application management.

.. list-table:: General endpoint summary
   :header-rows: 1

   * - Area
     - Base Path
     - Description
   * - Authentication
     - ``/api/v1/auth``
     - Login, refresh, and logout
   * - Users
     - ``/api/v1/users``
     - User account management
   * - System
     - ``/api/v1/system``
     - System information, metrics, configuration
   * - System Update
     - ``/api/v1/system/update``
     - Firmware update upload and installation
   * - Time Series (VictoriaMetrics)
     - ``/api/v1/vmetrics``
     - Metric queries, export, and management
   * - I/O
     - ``/api/v1/io``
     - I/O units, signals, ports, connections
   * - Modbus Server
     - ``/api/v1/io/endpoints/modbus``
     - Modbus TCP/RTU endpoint configuration
   * - Firewall
     - ``/api/v1/firewall``
     - Firewall rules and internet sharing
   * - Licensing
     - ``/api/v1/licensing``
     - Software license management
   * - Apps
     - ``/api/v1/apps``
     - Application control
   * - Alerting
     - ``/api/v1/alerting``
     - Alert signals, rules, and destinations
   * - Ping
     - ``/api/v1/ping``
     - Health check
   * - Storage
     - ``/api/v1/storage``
     - Storage usage and Docker management


Authentication & Authorization
------------------------------

Token Types
-----------

The system uses **JSON Web Tokens (JWT)** signed with the **EdDSA** algorithm.

.. list-table:: Token types
   :header-rows: 1

   * - Token
     - Lifetime
     - Purpose
   * - Access Token
     - 3 minutes
     - Authorizes API requests
   * - Refresh Token
     - 30 days
     - Obtains new access tokens


Authentication Mechanisms
-------------------------

Three mechanisms are accepted, in order of precedence:

1. ``x-access-token`` HTTP header
2. ``Authorization: Bearer <token>`` HTTP header
3. ``access_token`` query parameter

.. code-block:: http

   GET /api/v1/system/information HTTP/1.1
   Host: 192.168.1.100:1990
   Authorization: Bearer eyJhbGciOiJFZERTQSIs...

Authorization Scopes
--------------------

.. list-table:: Permission levels
   :header-rows: 1

   * - Function
     - Required Role
     - Description
   * - ``withSysAdminAuth``
     - SystemAdministrator
     - Required for most write operations and sensitive reads
   * - ``withAuth``
     - Any authenticated user
     - Used for session management (close own session)
   * - *(none)*
     - *(none)*
     - Public endpoints (ping, system clock read, system metrics, login)

User Roles
----------

.. list-table:: Roles
   :header-rows: 1

   * - Role (numeric)
     - Role name (string in JWT)
     - Privileges
   * - 0
     - ``SystemAdministrator``
     - Full access to all endpoints
   * - 1
     - ``GlobalAppAdministrator``
     - Read access to public endpoints
   * - 2
     - ``AppAdministrator``
     - Read access to public endpoints
   * - 3
     - ``AppUser``
     - Read access to public endpoints

Session Lifecycle
-----------------

.. code-block:: text

   +------------------+          +------------------+          +------------------+
   |   Login Request  |          |  Access Token    |          |   Refresh Token  |
   |   (credentials)  |---------->|  (3 min TTL)     |---------->|  (30 day TTL)   |
   +------------------+          +------------------+          +------------------+
                                                   |
                                                   v
                                          +------------------+
                                          |   Logout / Expire |
                                          +------------------+


API Design Patterns
-------------------

RESTful Collection Resources
----------------------------

Array-based resources follow a consistent CRUD pattern:

.. list-table:: Collection operations
   :header-rows: 1

   * - Method
     - Path
     - Description
   * - ``GET``
     - ``{basePath}``
     - List all resources
   * - ``GET``
     - ``{basePath}/<uuid>``
     - Get a single resource by UUID
   * - ``POST``
     - ``{basePath}``
     - Create a new resource (server generates UUID)
   * - ``PUT``
     - ``{basePath}/<uuid>``
     - Modify an existing resource
   * - ``DELETE``
     - ``{basePath}/<uuid>``
     - Remove a resource
   * - ``PUT``
     - ``{basePath}/disable/<uuid>``
     - Enable/disable a resource (where ``allowDisable`` is true)
   * - ``PUT``
     - ``{basePath}/<uuid>/up``
     - Move resource up in ordered list (where ``allowMoveUpDown`` is true)
   * - ``PUT``
     - ``{basePath}/<uuid>/down``
     - Move resource down in ordered list (where ``allowMoveUpDown`` is true)
   * - ``PUT``
     - ``{basePath}``
     - Replace all resources (where ``allowReplaceAll`` is true)

Configuration Object Resources
------------------------------

Configuration objects expose read/write of their property map:

.. list-table:: Config object operations
   :header-rows: 1

   * - Method
     - Path
     - Description
   * - ``GET``
     - ``{basePath}``
     - Read all configuration properties
   * - ``POST``
     - ``{basePath}``
     - Write configuration properties

UUID Generation
---------------

All server-side UUIDs are generated as 32-character hex strings (no dashes)
using ``CryptoUtils.createUuidWithoutDashes()``.

Pagination
----------

No pagination is implemented. All collection endpoints return the full
array. For large collections (e.g., time-series), the client is expected
to filter via query parameters.

Body Size Limits
----------------

.. list-table:: Limits
   :header-rows: 1

   * - Endpoint category
     - Max body size
   * - General REST resources
     - 1 MB (1,048,576 bytes)
   * - Unit config write / icon upload
     - 4 MB (4,194,304 bytes)
   * - Unit icon upload
     - 128 KB (131,072 bytes)
   * - License import
     - 100 KB (102,400 bytes)
   * - Unit name
     - 256 bytes
   * - OpenVPN config
     - 64 KB (65,536 bytes)
   * - Firewall internet sharing
     - 100 KB (102,400 bytes)
   * - Auth (login/refresh/logout)
     - 1 KB (1,000 bytes)
   * - SMS
     - 4 KB (4,096 bytes)
   * - System clock
     - 128 bytes
   * - App control
     - 10 bytes
   * - Update chunk
     - 4 MB (4,194,304 bytes)
   * - Update upload start
     - 1 KB (1,000 bytes)


Resource Sections
-----------------

Ping
----

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/ping``
     - None
     - Health check

Request
~~~~~~~

No body or parameters required.

Response
~~~~~~~~

.. code-block:: http

   HTTP/1.1 200 OK


Authentication
--------------

Login
~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/auth/login``
     - None
     - Authenticate and obtain tokens

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``username``
     - string
     - Yes
     - Login name of the user
   * - ``password``
     - string
     - Yes
     - Plain-text password (hashed server-side with SHA-256)

.. code-block:: json

   {
       "username": "hubadmin",
       "password": "secret123"
   }

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "accessToken": "eyJhbGciOiJFZERTQSIs...",
       "tokenType": "Bearer",
       "refreshToken": "a3f1c2e4b5d6478990abcdef12345678"
   }

**400 Bad Request** – body too large or missing fields

**401 Unauthorized** – invalid credentials


Refresh Access Token
~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/auth/refresh``
     - None
     - Exchange refresh token for new access token

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``refreshToken``
     - string
     - Yes
     - The refresh token obtained at login

.. code-block:: json

   {
       "refreshToken": "a3f1c2e4b5d6478990abcdef12345678"
   }

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "accessToken": "eyJhbGciOiJFZERTQSIs...",
       "tokenType": "Bearer"
   }

**400 Bad Request** – body too large or missing ``refreshToken``

**401 Unauthorized** – unknown refresh token

**500 Internal Server Error** – failed to create new access token


Logout
~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/auth/logout``
     - None
     - Invalidate a session by refresh token

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``refreshToken``
     - string
     - Yes
     - The refresh token to invalidate

.. code-block:: json

   {
       "refreshToken": "a3f1c2e4b5d6478990abcdef12345678"
   }

Response
^^^^^^^^

**200 OK** – session removed

**400 Bad Request** – body too large or missing ``refreshToken``

**401 Unauthorized** – unknown refresh token


Close Session
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``DELETE``
     - ``/api/v1/sessions/<sessionId>``
     - Any authenticated user
     - Close a specific session (must be own session)

Path Parameters
^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1

   * - Parameter
     - Type
     - Description
   * - ``sessionId``
     - string (UUID)
     - The session UUID to close

Response
^^^^^^^^

**200 OK** – session closed

**401 Unauthorized** – no valid access token

**403 Forbidden** – token valid but ``sid`` claim does not match ``sessionId``

**404 Not Found** – session does not exist


Users
-----

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/users``
     - SysAdmin
     - List all users
   * - ``GET``
     - ``/api/v1/users/<uuid>``
     - SysAdmin
     - Get a single user
   * - ``POST``
     - ``/api/v1/users``
     - SysAdmin
     - Create a user
   * - ``PUT``
     - ``/api/v1/users/<uuid>``
     - SysAdmin
     - Modify a user
   * - ``PUT``
     - ``/api/v1/users/disable/<uuid>``
     - SysAdmin
     - Enable/disable a user
   * - ``DELETE``
     - ``/api/v1/users/<uuid>``
     - SysAdmin
     - Delete a user

User Schema
~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto-generated
     - Unique identifier (32-char hex, no dashes)
   * - ``loginName``
     - string
     - Yes (create)
     - Unique login name
   * - ``fullName``
     - string
     - Yes (create)
     - Display name
   * - ``password``
     - string
     - Yes (create) / optional (modify)
     - Plain-text password; stored as SHA-256 hash. If omitted on modify, existing password is preserved.
   * - ``passwordConfirm``
     - string
     - No
     - Client-side confirmation; stripped server-side
   * - ``role``
     - int
     - Yes (create)
     - One of: 0 (SystemAdministrator), 1 (GlobalAppAdministrator), 2 (AppAdministrator), 3 (AppUser)
   * - ``disabled``
     - bool
     - Auto-set
     - Whether the account is disabled (default: false)

Note: The ``password`` field is **never** returned in GET responses.

Create User
~~~~~~~~~~~

.. code-block:: json
   :caption: POST /api/v1/users – Request

   {
       "loginName": "jdoe",
       "fullName": "John Doe",
       "password": "Str0ngP4ss!",
       "role": 3
   }

.. code-block:: text
   :caption: Response (200 OK)

   7a3f1c2e4b5d6478990abcdef12345678

**409 Conflict** – ``loginName`` already exists

**400 Bad Request** – missing required fields

Modify User
~~~~~~~~~~~

.. code-block:: json
   :caption: PUT /api/v1/users/7a3f1c2e4b5d6478990abcdef12345678 – Request

   {
       "loginName": "jdoe",
       "fullName": "John Q. Doe",
       "password": "NewP4ss!",
       "role": 2,
       "disabled": false
   }

.. code-block:: text
   :caption: Response (200 OK)

   (empty body)

**400 Bad Request** – invalid data

**404 Not Found** – user UUID not found

Delete User
~~~~~~~~~~~

The ``hubadmin`` account cannot be deleted.

.. code-block:: text
   :caption: Response (200 OK)

   (empty body)

**400 Bad Request** – attempting to delete protected ``hubadmin`` account

**404 Not Found** – user UUID not found


System
------

Information
~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/information``
     - None
     - Get system identification info

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "systemClock": 1700000000,
       "uptime": 123456,
       "osVersion": "3.2.1 (Standard)",
       "osLicenseValid": 1,
       "osLicenseExpiryDate": "2025-12-31T23:59:59.000Z",
       "ipAddresses": [
           "192.168.1.100 (eth0)",
           "172.16.0.5 (eth1)",
           "192.168.123.1 (usb0)"
       ]
   }

Metrics
~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/metrics``
     - None
     - Get real-time system metrics

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "totalCpuUsage": 23.5,
       "brokenDownCpuUsage": {
           "user": 12.1,
           "system": 5.3,
           "idle": 80.1,
           "iowait": 1.2,
           "irq": 0.3,
           "softirq": 1.5
       },
       "systemLoad": 0.42,
       "memoryTotal": 512000000,
       "memoryAvailable": 310000000,
       "storageTotal": 7500000000,
       "storageAvailable": 5200000000,
       "storageBytesReadPerSecond": 1024,
       "storageBytesWrittenPerSecond": 2048,
       "eth0Rx": 1500,
       "eth0Tx": 800,
       "eth1Rx": 0,
       "eth1Tx": 0,
       "wlan0Rx": 0,
       "wlan0Tx": 0,
       "wwan0Rx": 0,
       "wwan0Tx": 0
   }

Processes
~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/processes``
     - SysAdmin
     - List running processes

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   [
       {"pid": 1, "name": "systemd", "cpu": 0.1, "memory": 4500},
       {"pid": 234, "name": "smac-server", "cpu": 2.3, "memory": 45000}
   ]

Journal
~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/journal/recent``
     - SysAdmin
     - Get recent system journal messages

Response
^^^^^^^^

**200 OK** – plain text, newline-separated log messages

.. code-block:: text

   Jan 15 10:23:01 smac-server[234]: System started
   Jan 15 10:23:02 smac-server[234]: MQTT broker connected


Clock
~~~~~

.. list-table:: Clock endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/clock``
     - None
     - Get current system clock (epoch seconds)
   * - ``POST``
     - ``/api/v1/system/clock``
     - SysAdmin
     - Set system clock
   * - ``POST``
     - ``/api/v1/system/hardware-clocks``
     - SysAdmin
     - Synchronize hardware clocks to system time

Set Clock Request
^^^^^^^^^^^^^^^^^

.. code-block:: text
   :caption: POST /api/v1/system/clock – Body (plain text, < 128 bytes)

   1700000000

The value must be a valid epoch timestamp (> 1234567890, i.e., after 2009).

**400 Bad Request** – body too large or value not a valid epoch

**200 OK** – clock set, hardware clocks synchronized, licenses reloaded


Wireless Networks
~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/networking/available-wireless-networks``
     - SysAdmin
     - Scan and list available Wi-Fi networks

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   [
       {"ssid": "MyNetwork", "signalStrength": -45, "channel": 6, "encryption": "WPA2"},
       {"ssid": "GuestWiFi", "signalStrength": -70, "channel": 11, "encryption": "WPA3"}
   ]


Storage
~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/storage/usage``
     - SysAdmin
     - Get detailed storage partition usage

Response
^^^^^^^^

**200 OK** – JSON array of partition usage objects


Device Identification
~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/system/device/identify``
     - SysAdmin
     - Trigger LED blink for physical device identification

Response
^^^^^^^^

**200 OK** – LEDs blink for 10 seconds


Reboot
~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/system/reboot``
     - SysAdmin
     - Reboot the device

Response
^^^^^^^^

**200 OK** – reboot initiated (response may not arrive before shutdown)


Email Test
~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/system/mail/test``
     - SysAdmin
     - Send a test email using provided SMTP settings

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``enabled``
     - bool
     - Yes
     - SMTP enabled
   * - ``serverAddress``
     - string
     - Yes
     - SMTP server hostname
   * - ``serverPort``
     - int
     - Yes
     - SMTP server port
   * - ``encryptionMode``
     - int
     - Yes
     - 0: none, 1: SSL, 2: TLS
   * - ``authenticationMethod``
     - int
     - Yes
     - 0: none, 1: PLAIN, 2: LOGIN, 3: CRAM-MD5
   * - ``username``
     - string
     - No
     - SMTP username
   * - ``password``
     - string
     - No
     - SMTP password
   * - ``senderAddress``
     - string
     - Yes
     - From address
   * - ``senderName``
     - string
     - No
     - From display name
   * - ``recipient``
     - string
     - Yes
     - Test recipient address
   * - ``subject``
     - string
     - Yes
     - Email subject
   * - ``body``
     - string
     - Yes
     - Email body

.. code-block:: json

   {
       "enabled": true,
       "serverAddress": "smtp.example.com",
       "serverPort": 587,
       "encryptionMode": 2,
       "authenticationMethod": 2,
       "username": "alerts@example.com",
       "password": "secret",
       "senderAddress": "alerts@example.com",
       "senderName": "SIINEOS Alerts",
       "recipient": "admin@example.com",
       "subject": "Test Alert",
       "body": "This is a test email from SIINEOS."
   }

Response
^^^^^^^^

**200 OK** – email sent successfully

**400 Bad Request** – email failed to send


SMS
~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/sms``
     - SysAdmin
     - Send an SMS message

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``recipientNumbers``
     - string
     - Yes
     - Comma-separated phone numbers
   * - ``text``
     - string
     - Yes
     - SMS text content

.. code-block:: json

   {
       "recipientNumbers": "+4915123456789,+491609876543",
       "text": "Alert: High temperature on sensor T1"
   }

Response
^^^^^^^^

**200 OK** – SMS sent

**400 Bad Request** – missing/empty fields or body too large

**500 Internal Server Error** – modem/SMS send failed


OpenVPN
~~~~~~~

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/system/openvpn/client/config``
     - SysAdmin
     - Upload OpenVPN client configuration

Request
^^^^^^^

Body: raw OpenVPN configuration file content (text, max 64 KB).

Response
^^^^^^^^

**200 OK** – config written and service restarted

**400 Bad Request** – body too large

**500 Internal Server Error** – failed to write config file


System Configuration
~~~~~~~~~~~~~~~~~~~~

.. list-table:: Config object endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/system/config/<object>``
     - SysAdmin
     - Read configuration object
   * - ``POST``
     - ``/api/v1/system/config/<object>``
     - SysAdmin
     - Write configuration object

Available ``<object>`` values:

.. list-table::
   :header-rows: 1

   * - Object ID
     - Description
   * - ``datetime``
     - Timezone, NTP server, browser auto-sync
   * - ``eth0``
     - Ethernet 1 settings (mode, IP, DHCP server)
   * - ``eth1``
     - Ethernet 2 settings
   * - ``wlan0``
     - Wi-Fi settings
   * - ``wwan0``
     - Cellular (WWAN) settings
   * - ``device``
     - Device name, hostname, location
   * - ``communication-leds``
     - Communication LED targets
   * - ``debugging``
     - Debug/trace logging, filter rules
   * - ``services``
     - SSH, VictoriaMetrics, Docker, MQTT broker, memory monitor
   * - ``tls``
     - TLS certificates (CA, device cert, private key)
   * - ``smtp``
     - SMTP email settings

Example – Read device settings:

.. code-block:: json
   :caption: GET /api/v1/system/config/device – Response (200 OK)

   {
       "name": "hub-gm100-abc123",
       "location": "Building 1, Room 234"
   }

Example – Write device settings:

.. code-block:: json
   :caption: POST /api/v1/system/config/device – Request

   {
       "name": "Central Gateway",
       "location": "Rooftop Antenna"
   }


System Update
-------------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``POST``
     - ``/api/v1/system/update/upload/start``
     - SysAdmin
     - Begin uploading an update bundle
   * - ``POST``
     - ``/api/v1/system/update/upload/chunk``
     - SysAdmin
     - Upload a data chunk
   * - ``GET``
     - ``/api/v1/system/update/install/state``
     - SysAdmin
     - Query installation progress
   * - ``GET``
     - ``/api/v1/system/update/error``
     - SysAdmin
     - Get last update error

Upload Start
~~~~~~~~~~~~

Request
^^^^^^^

.. list-table:: Body fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``bytesToReceive``
     - int
     - Yes
     - Total expected file size in bytes (1 KB – 256 MB)

.. code-block:: json

   {
       "bytesToReceive": 134217728
   }

Response
^^^^^^^^

**200 OK** – ready to accept chunks

**400 Bad Request** – body too large, size < 1 KB, or size > 256 MB

**500 Internal Server Error** – failed to open output file

Upload Chunk
~~~~~~~~~~~~

Request: raw binary data (max 4 MB per chunk).

Response
^^^^^^^^

**200 OK** – body contains the current byte offset as a string

.. code-block:: text
   :caption: Response body (200 OK)

   5242880

When the total received equals ``bytesToReceive``, the update installation
begins automatically.

**400 Bad Request** – chunk too large, file not open, or size limit exceeded

Installation State
~~~~~~~~~~~~~~~~~~

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "message": "Installing system update…",
       "progress": 45
   }

Error
~~~~~

Response
^^^^^^^^

**200 OK** – body contains the error message string (or empty if no error)


Time Series Database (VictoriaMetrics)
--------------------------------------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/vmetrics/timeseries``
     - SysAdmin
     - List available time series
   * - ``GET``
     - ``/api/v1/vmetrics/export/csv``
     - SysAdmin
     - Export metric data as CSV (streamed)
   * - ``POST``
     - ``/api/v1/vmetrics/timeseries/delete``
     - SysAdmin
     - Delete time series
   * - ``POST``
     - ``/api/v1/vmetrics/reset``
     - SysAdmin
     - Reset (restart) VictoriaMetrics service

List Time Series
~~~~~~~~~~~~~~~~

.. list-table:: Query parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``start``
     - string (epoch ms)
     - Yes
     - Start of time range (13+ digits)
   * - ``end``
     - string (epoch ms)
     - Yes
     - End of time range (13+ digits)

.. code-block:: http

   GET /api/v1/vmetrics/timeseries?start=1700000000000&end=1700086400000
   Authorization: Bearer <token>

Response
^^^^^^^^

**200 OK** – JSON array of series metadata

.. code-block:: json

   [
       {"name": "eth0_rx[unit=eth0][signal=rx_bytes](avg)", "hash": "a1b2c3d4e5f6"},
       {"name": "cpu_usage[unit=hub01][signal=load](avg)", "hash": "f6e5d4c3b2a1"}
   ]

**202 Accepted** – labels are still being fetched; retry later

**400 Bad Request** – missing or invalid ``start``/``end``

Export CSV
~~~~~~~~~~

.. list-table:: Query parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``start``
     - string (epoch ms)
     - Yes
     - Start of time range
   * - ``end``
     - string (epoch ms)
     - Yes
     - End of time range
   * - ``step``
     - int (seconds)
     - No
     - Query step size (default: 60)
   * - ``rollup``
     - string
     - No
     - Rollup functions: comma-separated from ``min``, ``max``, ``avg``, ``sum``, ``cnt``
   * - ``series``
     - string
     - Yes
     - Comma-separated list of series hashes
   * - ``decimalSeparator``
     - string
     - No
     - Decimal separator for output (default: ``.``)
   * - ``dateTimeFormat``
     - string
     - No
     - One of: ``timestamp``, ``local``, ``utc``, ``localized``, ``iso`` (default: ``timestamp``)
   * - ``dateTimeLocale``
     - string
     - No
     - Locale code, e.g. ``de_DE``, ``C`` (default: ``C``)

.. code-block:: http

   GET /api/v1/vmetrics/export/csv?start=1700000000000&end=1700086400000&step=60&rollup=avg&series=a1b2c3d4e5f6,f6e5d4c3b2a1&decimalSeparator=,
   Authorization: Bearer <token>

Response
^^^^^^^^

**200 OK** – streamed CSV with ``Content-Disposition: attachment``

.. code-block:: text
   :caption: Response headers and body

   Content-Type: text/plain
   Content-Disposition: attachment; filename="SIINEOS-VictoriaMetrics-Export_1700000000000_1700086400000.csv"

   Timestamp;eth0_rx[unit=eth0][signal=rx_bytes](avg);cpu_usage[unit=hub01][signal=load](avg)
   1700000000;1523.45;0.42
   1700000060;1498.12;0.38
   ...

**400 Bad Request** – invalid parameters

**401 Unauthorized** – missing/invalid token

Delete Time Series
~~~~~~~~~~~~~~~~~~

.. code-block:: json
   :caption: POST /api/v1/vmetrics/timeseries/delete – Request

   ["a1b2c3d4e5f6", "f6e5d4c3b2a1"]

Response
^^^^^^^^

**200 OK** – series deleted (or proxied status from VictoriaMetrics)

**400 Bad Request** – empty or missing array

**404 Not Found** – a series hash does not exist

**500 Internal Server Error** – HTTP error from VictoriaMetrics

Reset VictoriaMetrics
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text
   :caption: POST /api/v1/vmetrics/reset – Response (200 OK)

   (empty body)

**500 Internal Server Error** – service restart failed


I/O Management
--------------

Overview
--------

I/O units represent physical or virtual data sources (sensors, fieldbus
nodes, Modbus devices, synthetic signals). Each unit contains **signals**
(measurement points) and optionally **ports** (physical connection points).

.. list-table:: I/O endpoint summary
   :header-rows: 1

   * - Area
     - Base Path
   * - Units
     - ``/api/v1/io/units``
   * - Signals
     - ``/api/v1/io/units/<unitUuid>/signals``
   * - Ports
     - ``/api/v1/io/units/<unitUuid>/ports/<portUuid>``
   * - Connections
     - ``/api/v1/io/connections``
   * - Synthetic Signals
     - ``/api/v1/io/synthetic/signals``
   * - Signal Processing (default)
     - ``/api/v1/io/signal-processing/config/default``
   * - Modbus Server
     - ``/api/v1/io/endpoints/modbus``


I/O Units
---------

.. list-table:: Unit endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/units``
     - SysAdmin
     - Get UUIDs of all I/O units
   * - ``POST``
     - ``/api/v1/io/units``
     - SysAdmin
     - Create a new unit for a provider
   * - ``GET``
     - ``/api/v1/io/units/<unitUuid>``
     - SysAdmin
     - Get information about a specific I/O unit
   * - ``DELETE``
     - ``/api/v1/io/units/<uuid>``
     - SysAdmin
     - Remove a unit
   * - ``GET``
     - ``/api/v1/io/units/<uuid>/config``
     - SysAdmin
     - Read full unit configuration
   * - ``PUT``
     - ``/api/v1/io/units/<uuid>/config``
     - SysAdmin
     - Write full unit configuration
   * - ``GET``
     - ``/api/v1/io/units/<uuid>/config/<objectId>``
     - SysAdmin
     - Read a specific config sub-object
   * - ``PUT``
     - ``/api/v1/io/units/<uuid>/config/<objectId>``
     - SysAdmin
     - Write a specific config sub-object
   * - ``PUT``
     - ``/api/v1/io/units/<uuid>/name``
     - SysAdmin
     - Set unit display name
   * - ``PUT``
     - ``/api/v1/io/units/<uuid>/upstream``
     - SysAdmin
     - Set upstream unit/port
   * - ``GET``
     - ``/api/v1/io/units/<uuid>/icon``
     - None
     - Get unit icon (binary)
   * - ``PUT``
     - ``/api/v1/io/units/<uuid>/icon``
     - SysAdmin
     - Set unit icon (binary, max 128 KB)
   * - ``DELETE``
     - ``/api/v1/io/units/<uuid>/icon``
     - SysAdmin
     - Clear custom icon
   * - ``POST``
     - ``/api/v1/io/units/<uuid>/control/<action>``
     - SysAdmin
     - Provider-specific control action

Create Unit
~~~~~~~~~~~

.. code-block:: text
   :caption: POST /api/v1/io/units – Body (plain text: provider identifier)

   modbus_client

Response
^^^^^^^^

**200 OK** – body contains the new unit UUID

.. code-block:: text

   907cc14b665a47f2963907f344f7bb73

**400 Bad Request** – provider identifier not found

Remove Unit
~~~~~~~~~~~

.. code-block:: text
   :caption: DELETE /api/v1/io/units/907cc14b665a47f2963907f344f7bb73 – Response (200 OK)

   (empty body)

**404 Not Found** – unit not found

Read Unit Config
~~~~~~~~~~~~~~~~

Response
^^^^^^^^

**200 OK** – full serialized configuration including all config objects and items.

.. code-block:: json

   {
       "objects": {
           "general": {
               "name": "My Modbus Device",
               "state": true,
               "uuid": "907cc14b665a47f2963907f344f7bb73",
               "providerId": "modbus_client",
               "upstreamUnit": "",
               "upstreamPort": "",
               "signals": []
           },
           "data": {
               "customIcon": ""
           }
       }
   }

Set Unit Name
~~~~~~~~~~~~~

.. code-block:: text
   :caption: PUT /api/v1/io/units/<uuid>/name – Body (plain text, max 256 bytes)

   Temperature Sensor Bank A

Set Unit Upstream
~~~~~~~~~~~~~~~~~

.. code-block:: json
   :caption: PUT /api/v1/io/units/<uuid>/upstream – Request

   {
       "unit": "a1b2c3d4e5f6478990abcdef12345678",
       "port": "port-uuid-001"
   }

Unit Icon
~~~~~~~~~

GET returns the raw image bytes (or 404 if no icon set).

PUT accepts raw image bytes (max 128 KB).

.. code-block:: http
   :caption: GET /api/v1/io/units/<uuid>/icon – Response

   HTTP/1.1 200 OK
   (binary image data)


I/O Signals
-----------

.. list-table:: Signal endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/units/<unitUuid>/signals``
     - SysAdmin
     - Get UUIDs of all signals
   * - ``POST``
     - ``/api/v1/io/units/<unitUuid>/signals``
     - SysAdmin
     - Add a new signal
   * - ``PUT``
     - ``/api/v1/io/units/<unitUuid>/signals``
     - SysAdmin
     - Add multiple signals
   * - ``GET``
     - ``/api/v1/io/units/<unitUuid>/signals/<signalUuid>``
     - SysAdmin
     - Get information about a specific signal
   * - ``DELETE``
     - ``/api/v1/io/units/<unitUuid>/signals?uuids=<uuid1>,<uuid2>``
     - SysAdmin
     - Remove signals
   * - ``POST``
     - ``/api/v1/io/units/<unitUuid>/signals/<signalUuid>/duplicate``
     - SysAdmin
     - Duplicate a signal
   * - ``GET``
     - ``/api/v1/io/units/<unitUuid>/signals/<signalUuid>/config``
     - SysAdmin
     - Read signal configuration
   * - ``PUT``
     - ``/api/v1/io/units/<unitUuid>/signals/<signalUuid>/config``
     - SysAdmin
     - Write signal configuration
   * - ``PUT``
     - ``/api/v1/io/units/<unitUuid>/signals/config?uuids=<uuid1>,<uuid2>``
     - SysAdmin
     - Write config for multiple signals
   * - ``PUT``
     - ``/api/v1/io/units/<unitUuid>/signals/clear``
     - SysAdmin
     - Remove all signals from unit
   * - ``PUT``
     - ``/api/v1/io/units/<unitUuid>/signals/reset``
     - SysAdmin
     - Reset specific signals
   * - ``POST``
     - ``/api/v1/io/units/<unitUuid>/signals/<signalUuid>/control/<action>``
     - SysAdmin
     - Provider-specific signal control

Signal Configuration Schema
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table:: Signal config fields
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``name``
     - string
     - Display name
   * - ``uuid``
     - string
     - System ID (read-only)
   * - ``enabled``
     - bool
     - Signal enabled
   * - ``vmetricsPushEnabled``
     - bool
     - Record to VictoriaMetrics
   * - ``vmetricsPushInterval``
     - int
     - Recording interval in seconds (1–3600)
   * - ``useCustomIdentifier``
     - bool
     - Use custom identifier instead of auto-generated
   * - ``customIdentifier``
     - string
     - Custom identifier string
   * - ``group``
     - string
     - Group/category label
   * - ``dataSeriesSet``
     - string
     - Data series set label
   * - ``siPrefix``
     - int
     - SI prefix (0=none, 1=G, 2=M, 3=k, 4=h, 6=d, 7=c, 8=m, 9=µ, 10=n, 11=p)
   * - ``measurementUnit``
     - string
     - Unit string (e.g., "DegreeCelsius", "Watt", "Volt")
   * - ``decimals``
     - int
     - Number of decimal places (0–5)
   * - ``customDataType``
     - int
     - Override data type (0=keep, 1=bool, 5=float, 6=double, 7–10=int types, 13–16=small int)
   * - ``visualizationType``
     - string
     - "none", "gauge", "led", "counter"
   * - ``color``
     - string
     - Color for visualization
   * - ``minimumValue``
     - float
     - Minimum expected value
   * - ``maximumValue``
     - float
     - Maximum expected value

Add Single Signal
~~~~~~~~~~~~~~~~~

.. code-block:: text
   :caption: POST /api/v1/io/units/<unitUuid>/signals – Response (200 OK)

   new-signal-uuid-here

Add Multiple Signals
~~~~~~~~~~~~~~~~~~~~

.. code-block:: json
   :caption: PUT /api/v1/io/units/<unitUuid>/signals – Request

   [
       {"name": "Temperature", "measurementUnit": "DegreeCelsius"},
       {"name": "Humidity", "measurementUnit": "%"}
   ]

.. code-block:: json
   :caption: Response (200 OK)

   ["uuid-1", "uuid-2"]

Remove Signals
~~~~~~~~~~~~~~

.. code-block:: http
   :caption: DELETE request with query parameter

   DELETE /api/v1/io/units/<unitUuid>/signals?uuids=uuid1,uuid2,uuid3
   Authorization: Bearer <token>

Duplicate Signal
~~~~~~~~~~~~~~~~

.. code-block:: json
   :caption: POST /api/v1/io/units/<unitUuid>/signals/<signalUuid>/duplicate – Request

   {
       "name": "Temperature (Copy)"
   }

.. code-block:: text
   :caption: Response (200 OK)

   new-duplicated-signal-uuid

Reset Signals
~~~~~~~~~~~~~

.. code-block:: json
   :caption: PUT /api/v1/io/units/<unitUuid>/signals/reset – Request

   ["signal-uuid-1", "signal-uuid-2"]

Signal Control
~~~~~~~~~~~~~~

Provider-specific actions. The ``action`` path segment and body are
provider-defined.

.. code-block:: http

   POST /api/v1/io/units/<unitUuid>/signals/<signalUuid>/control/calibrate
   Authorization: Bearer <token>
   Content-Type: application/octet-stream

   (binary or text payload, max 4 MB)


I/O Ports
---------

.. list-table:: Port endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/units/<unitUuid>/ports/<portUuid>/config``
     - SysAdmin
     - Read port configuration
   * - ``POST``
     - ``/api/v1/io/units/<unitUuid>/ports/<portUuid>/config``
     - SysAdmin
     - Write port configuration

Response
^^^^^^^^

**200 OK** – serialized port config object

**404 Not Found** – unit or port not found


I/O Connections
---------------

Connections define signal-to-signal data flow with optional signal
processing chains.

.. list-table:: Connection endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/connections``
     - SysAdmin
     - List all connections
   * - ``GET``
     - ``/api/v1/io/connections/<uuid>``
     - SysAdmin
     - Get a single connection
   * - ``POST``
     - ``/api/v1/io/connections``
     - SysAdmin
     - Create a connection
   * - ``PUT``
     - ``/api/v1/io/connections/<uuid>``
     - SysAdmin
     - Modify a connection
   * - ``PUT``
     - ``/api/v1/io/connections/disable/<uuid>``
     - SysAdmin
     - Enable/disable a connection
   * - ``DELETE``
     - ``/api/v1/io/connections/<uuid>``
     - SysAdmin
     - Delete a connection

Connection Schema
~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto
     - Unique identifier
   * - ``source.unit``
     - string
     - Yes
     - UUID of source I/O unit
   * - ``source.signal``
     - string
     - Yes
     - UUID of source signal
   * - ``destination.unit``
     - string
     - Yes
     - UUID of destination I/O unit
   * - ``destination.signal``
     - string
     - Yes
     - UUID of destination signal
   * - ``disabled``
     - bool
     - No
     - Whether the connection is disabled (default: false)
   * - ``signalProcessings``
     - object
     - No
     - Signal processing chain configuration

.. code-block:: json
   :caption: Example connection

   {
       "source": {"unit": "unit-aaa", "signal": "sig-111"},
       "destination": {"unit": "unit-bbb", "signal": "sig-222"},
       "disabled": false,
       "signalProcessings": {
           "preProcessingEnabled": true,
           "preProcessingExpression": "x * 2",
           "linearScalingEnabled": true,
           "linearScalingX1": 0,
           "linearScalingX2": 100,
           "linearScalingY1": 20,
           "linearScalingY2": 80
       }
   }


Signal Processing Chain
-----------------------

The default signal processing chain configuration is exposed at:

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/signal-processing/config/default``
     - SysAdmin
     - Read default processing chain template (read-only)

.. list-table:: Processing stages (in order)
   :header-rows: 1

   * - Stage
     - Config prefix
     - Description
   * - Pre-processing
     - ``preProcessing*``
     - Mathematical expression applied to raw input
   * - Linear scaling
     - ``linearScaling*``
     - Map input range [X1, X2] to output range [Y1, Y2]
   * - Delta
     - ``delta*``
     - Compute difference/ratio to previous value
   * - Limit
     - ``limit*``
     - Clamp value to [min, max]
   * - Threshold comparison
     - ``thresholdComparison*``
     - Convert to boolean based on comparison
   * - Comparator
     - ``comparator*``
     - Two-threshold comparator with debounce
   * - Edge detection
     - ``edgeDetection*``
     - Count rising/falling edges
   * - Time derivative
     - ``timeDerivative*``
     - Compute rate of change (Δx/s, Δx/min, Δx/h, Δx/d)
   * - Aggregation
     - ``aggregation*``
     - Rolling window: min, max, avg, oldest, sum, median
   * - Post-processing
     - ``postProcessing*``
     - Mathematical expression applied to final output


Synthetic Signals
-----------------

Virtual signals computed from other signals using arithmetic operations
or mathematical expressions.

.. list-table:: Synthetic signal endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/synthetic/signals``
     - SysAdmin
     - List all synthetic signals
   * - ``GET``
     - ``/api/v1/io/synthetic/signals/<uuid>``
     - SysAdmin
     - Get a single synthetic signal
   * - ``POST``
     - ``/api/v1/io/synthetic/signals``
     - SysAdmin
     - Create a synthetic signal
   * - ``PUT``
     - ``/api/v1/io/synthetic/signals/<uuid>``
     - SysAdmin
     - Modify a synthetic signal
   * - ``DELETE``
     - ``/api/v1/io/synthetic/signals/<uuid>``
     - SysAdmin
     - Delete a synthetic signal
   * - ``PUT``
     - ``/api/v1/io/synthetic/signals/disable/<uuid>``
     - SysAdmin
     - Enable/disable a synthetic signal
   * - ``PUT``
     - ``/api/v1/io/synthetic/signals``
     - SysAdmin
     - Replace all synthetic signals
   * - ``POST``
     - ``/api/v1/io/synthetic/signals/<srcUuid>/clone/<dstUuid>``
     - SysAdmin
     - Clone settings from one signal to another
   * - ``POST``
     - ``/api/v1/io/synthetic/signals/<uuid>/name/propagate``
     - SysAdmin
     - Propagate name to I/O signal
   * - ``POST``
     - ``/api/v1/io/synthetic/signals/<uuid>/name/update``
     - SysAdmin
     - Update name from I/O signal

Synthetic Signal Schema
~~~~~~~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto
     - Unique identifier
   * - ``name``
     - string
     - Yes
     - Display name
   * - ``enabled``
     - bool
     - No
     - Signal enabled (default: true)
   * - ``source1.unit``
     - string
     - Yes
     - UUID of source unit 1
   * - ``source1.signal``
     - string
     - Yes
     - UUID of source signal 1
   * - ``source2.unit``
     - string
     - Yes
     - UUID of source unit 2
   * - ``source2.signal``
     - string
     - Yes
     - UUID of source signal 2
   * - ``calculation``
     - string
     - Yes
     - Operation: ``+``, ``-``, ``*``, ``/``, ``&&``, ``||``, ``..``, ``+.``, ``++``, or a math expression

Calculation operators:

.. list-table::
   :header-rows: 1

   * - Value
     - Description
   * - ``+``
     - A + B
   * - ``-``
     - A - B
   * - ``*``
     - A × B
   * - ``/``
     - A ÷ B
   * - ``&&``
     - Logical AND (A && B)
   * - ``||``
     - Logical OR (A || B)
   * - ``..``
     - Latch: A sets to 1, B resets to 0
   * - ``+.``
     - Counter with reset: counts A, resets when B is true
   * - ``++``
     - Counter without reset: counts A
   * - *(expression)*
     - Math expression using variables A, B, P (previous value)

.. code-block:: json
   :caption: Example synthetic signal

   {
       "name": "Total Power",
       "enabled": true,
       "source1": {"unit": "unit-aaa", "signal": "sig-voltage"},
       "source2": {"unit": "unit-aaa", "signal": "sig-current"},
       "calculation": "*"
   }


Modbus Server Endpoint
----------------------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/io/endpoints/modbus/tcp``
     - SysAdmin
     - Read Modbus TCP settings
   * - ``POST``
     - ``/api/v1/io/endpoints/modbus/tcp``
     - SysAdmin
     - Write Modbus TCP settings
   * - ``GET``
     - ``/api/v1/io/endpoints/modbus/rtu``
     - SysAdmin
     - Read Modbus RTU settings
   * - ``POST``
     - ``/api/v1/io/endpoints/modbus/rtu``
     - SysAdmin
     - Write Modbus RTU settings
   * - ``GET``
     - ``/api/v1/io/endpoints/modbus/registers``
     - SysAdmin
     - List Modbus registers
   * - ``POST``
     - ``/api/v1/io/endpoints/modbus/registers``
     - SysAdmin
     - Add a register
   * - ``PUT``
     - ``/api/v1/io/endpoints/modbus/registers/<uuid>``
     - SysAdmin
     - Modify a register
   * - ``DELETE``
     - ``/api/v1/io/endpoints/modbus/registers/<uuid>``
     - SysAdmin
     - Remove a register

Modbus TCP Settings
~~~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Default
     - Description
   * - ``enabled``
     - bool
     - false
     - Enable Modbus TCP server
   * - ``networkAddress``
     - string
     - "0.0.0.0"
     - Listen address
   * - ``networkPort``
     - int
     - 502
     - Listen port (1–65535)

Modbus RTU Settings
~~~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Default
     - Description
   * - ``enabled``
     - bool
     - false
     - Enable Modbus RTU slave
   * - ``rtuBusInterface``
     - int
     - 0
     - 0: serial port, 1: RS485, 2: backplane bus
   * - ``rtuSerialPortName``
     - string
     - "ttyUSB0"
     - Serial port device name
   * - ``rtuBaudRate``
     - int
     - 115200
     - Baud rate
   * - ``rtuDataBits``
     - int
     - 8
     - Data bits (5–8)
   * - ``rtuParity``
     - int
     - 0
     - 0: none, 2: even, 3: odd, 4: space, 5: mark
   * - ``rtuStopBits``
     - int
     - 1
     - Stop bits (1, 2, 3=1.5)
   * - ``modbusId``
     - int
     - 1
     - Modbus slave ID (1–254)

Register Schema
~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``type``
     - int
     - Yes
     - Modbus register type (coil, discrete, input reg, holding reg)
   * - ``address``
     - int
     - Yes
     - Register address
   * - ``dataType``
     - int
     - No
     - Data type (unsigned/signed int, float, double, etc.)
   * - ``ioMode``
     - int
     - No
     - 0: signal→register, 1: register→signal
   * - ``signal.unit``
     - string
     - No
     - Target I/O unit UUID (when linked to signal)
   * - ``signal.signal``
     - string
     - No
     - Target signal UUID (when linked to signal)


Firewall
--------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/firewall/rules/incoming``
     - SysAdmin
     - List incoming rules
   * - ``POST``
     - ``/api/v1/firewall/rules/incoming``
     - SysAdmin
     - Add incoming rule
   * - ``PUT``
     - ``/api/v1/firewall/rules/incoming/<uuid>``
     - SysAdmin
     - Modify incoming rule
   * - ``PUT``
     - ``/api/v1/firewall/rules/incoming/<uuid>/up``
     - SysAdmin
     - Move rule up
   * - ``PUT``
     - ``/api/v1/firewall/rules/incoming/<uuid>/down``
     - SysAdmin
     - Move rule down
   * - ``DELETE``
     - ``/api/v1/firewall/rules/incoming/<uuid>``
     - SysAdmin
     - Delete rule
   * - *(same pattern for)*
     - ``/api/v1/firewall/rules/outgoing``
     - SysAdmin
     - Outgoing rules
   * - *(same pattern for)*
     - ``/api/v1/firewall/rules/ipforwarding``
     - SysAdmin
     - IP forwarding rules
   * - *(same pattern for)*
     - ``/api/v1/firewall/rules/portforwarding``
     - SysAdmin
     - Port forwarding rules
   * - ``GET``
     - ``/api/v1/firewall/internet/sharing``
     - SysAdmin
     - Get internet sharing settings
   * - ``POST``
     - ``/api/v1/firewall/internet/sharing``
     - SysAdmin
     - Set internet sharing settings

Firewall Rule Schema
~~~~~~~~~~~~~~~~~~~~

.. list-table:: Common rule fields
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``networkProtocol``
     - int
     - IP protocol number (6=TCP, 17=UDP, 0=any)
   * - ``inputInterface`` / ``outputInterface``
     - string
     - Interface name (e.g., "eth0", "wlan0", "wwan0")
   * - ``sourceAddress``
     - string
     - Source IP/CIDR (e.g., "192.168.1.0/24")
   * - ``destinationAddress``
     - string
     - Destination IP/CIDR
   * - ``destinationPorts``
     - string
     - Space-separated port list (e.g., "80 443 8080")
   * - ``action``
     - int
     - 0: accept, 1: drop (nftables statement type)
   * - ``enabled``
     - bool
     - Whether the rule is active

.. code-block:: json
   :caption: Example incoming rule

   {
       "networkProtocol": 6,
       "inputInterface": "eth0",
       "sourceAddress": "192.168.1.0/24",
       "destinationPorts": "443 8080",
       "action": 0,
       "enabled": true
   }

Internet Sharing
~~~~~~~~~~~~~~~~

.. code-block:: json
   :caption: GET /api/v1/firewall/internet/sharing – Response

   {
       "enabled": true,
       "inputInterfaces": ["eth0", "eth1"],
       "outputInterface": "wwan0"
   }

.. code-block:: json
   :caption: POST /api/v1/firewall/internet/sharing – Request

   {
       "enabled": true,
       "inputInterfaces": ["eth0"],
       "outputInterface": "wlan0"
   }

Known network interfaces: ``eth0``, ``eth1``, ``wlan0``, ``wwan0``,
``tun0``, ``docker0``.


Licensing
---------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/licensing/licenses``
     - None
     - List all installed licenses
   * - ``POST``
     - ``/api/v1/licensing/licenses``
     - SysAdmin
     - Import a license file
   * - ``DELETE``
     - ``/api/v1/licensing/licenses/<licenseId>``
     - SysAdmin
     - Remove a license

List Licenses
~~~~~~~~~~~~~

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   [
       {
           "id": "lic-abc123",
           "productId": "de.inhub.siineos",
           "productName": "SIINEOS Standard",
           "productVersion": "3.2",
           "productSize": "Standard",
           "valid": true,
           "validFrom": "2024-01-01T00:00:00.000Z",
           "validUntil": "2025-12-31T23:59:59.000Z",
           "licensee": "ACME Corp"
       }
   ]

Import License
~~~~~~~~~~~~~~

Request: raw license file content (max 100 KB).

Response
^^^^^^^^

**200 OK**

.. code-block:: json

   {
       "id": "lic-abc123",
       "productId": "de.inhub.siineos",
       "productName": "SIINEOS Standard",
       "productVersion": "3.2",
       "productSize": "Standard",
       "valid": true,
       "validFrom": "2024-01-01T00:00:00.000Z",
       "validUntil": "2025-12-31T23:59:59.000Z",
       "licensee": "ACME Corp"
   }

**400 Bad Request** – body too large

**409 Conflict** – license with same ID already installed

**406 Not Acceptable** – license is invalid (expired or bad signature)

**422 Unprocessable Entity** – failed to parse license file

Remove License
~~~~~~~~~~~~~~

**200 OK** – license removed

**404 Not Found** – license ID not found


Apps
----

.. list-table::
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``PUT``
     - ``/api/v1/apps/<appId>/control/<property>``
     - SysAdmin
     - Set an app control property

Path Parameters
~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Parameter
     - Values
     - Description
   * - ``appId``
     - string
     - Application identifier
   * - ``property``
     - ``enabled``, ``debug``, ``trace``
     - The control property to set

Request
^^^^^^^

Body: JSON boolean (max 10 bytes).

.. code-block:: json

   true

Response
^^^^^^^^

**200 OK** – property set

**400 Bad Request** – body too large or invalid JSON

**404 Not Found** – unknown control property


Alerting
--------

.. list-table:: Endpoints
   :header-rows: 1

   * - Method
     - Path
     - Auth
     - Description
   * - ``GET``
     - ``/api/v1/alerting/signals``
     - SysAdmin
     - List alert signals
   * - ``POST``
     - ``/api/v1/alerting/signals``
     - SysAdmin
     - Create an alert signal
   * - ``PUT``
     - ``/api/v1/alerting/signals/<uuid>``
     - SysAdmin
     - Modify an alert signal
   * - ``PUT``
     - ``/api/v1/alerting/signals/disable/<uuid>``
     - SysAdmin
     - Enable/disable an alert signal
   * - ``DELETE``
     - ``/api/v1/alerting/signals/<uuid>``
     - SysAdmin
     - Delete an alert signal
   * - *(same pattern for)*
     - ``/api/v1/alerting/destinations``
     - SysAdmin
     - Alert destinations CRUD
   * - *(same pattern for)*
     - ``/api/v1/alerting/rules``
     - SysAdmin
     - Alert rules CRUD

Alert Signal Schema
~~~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto
     - Unique identifier
   * - ``name``
     - string
     - Yes
     - Display name
   * - ``disabled``
     - bool
     - No
     - Signal disabled (default: false)
   * - ``source.unit``
     - string
     - Yes
     - I/O unit UUID
   * - ``source.signal``
     - string
     - Yes
     - I/O signal UUID
   * - ``evalMode``
     - string
     - Yes
     - One of: ``binary``, ``thresholds``, ``counter``, ``cycles``
   * - ``evalParams``
     - object
     - Yes
     - Mode-specific parameters (see below)
   * - ``severity``
     - string
     - No
     - One of: ``none``, ``low``, ``medium``, ``high``
   * - ``category``
     - string
     - No
     - Free-text category
   * - ``cycleTime``
     - int
     - Yes (counter/cycles)
     - Watch window in milliseconds
   * - ``stateTransition``
     - object
     - No
     - Transition delays

``evalParams`` by mode:

.. list-table:: Binary mode
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``lowActive``
     - bool
     - If true, 0 triggers alarm

.. list-table:: Thresholds mode
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``comparison``
     - string
     - ``above``, ``aboveOrEqual``, ``below``, ``belowOrEqual``, ``equal``, ``insideRange``, ``outsideRange``
   * - ``lowerThreshold``
     - double
     - Lower threshold value
   * - ``upperThreshold``
     - double
     - Upper threshold value (for range comparisons)

.. list-table:: Counter/Cycles mode
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``counterStepSize``
     - double
     - Expected increment (counter mode)
   * - ``pulseThreshold``
     - double
     - Pulse count threshold (cycles mode)

``stateTransition``:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``alarmDelayEnabled``
     - bool
     - Enable alarm delay
   * - ``alarmDelay``
     - int
     - Delay in seconds before transitioning to ALARM
   * - ``okDelayEnabled``
     - bool
     - Enable OK delay
   * - ``okDelay``
     - int
     - Delay in seconds before transitioning to OK

.. code-block:: json
   :caption: Example alert signal (thresholds)

   {
       "name": "High Temperature",
       "source": {"unit": "unit-aaa", "signal": "sig-temp1"},
       "evalMode": "thresholds",
       "evalParams": {
           "comparison": "above",
           "lowerThreshold": 80
       },
       "severity": "high",
       "category": "Temperature",
       "stateTransition": {
           "alarmDelayEnabled": true,
           "alarmDelay": 30,
           "okDelayEnabled": false
       }
   }

Alert Destination Schema
~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table:: Common fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto
     - Unique identifier
   * - ``name``
     - string
     - Yes
     - Display name
   * - ``disabled``
     - bool
     - No
     - Destination disabled (default: false)
   * - ``type``
     - string
     - Yes
     - One of: ``email``, ``sms``, ``webhook``, ``app``, ``iosignal``, ``mqtt``, ``vmetrics``

Type-specific fields:

.. list-table:: Email
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``email.recipients``
     - string
     - Comma-separated email addresses
   * - ``email.subject``
     - string
     - Subject with placeholders
   * - ``email.body``
     - string
     - Body with placeholders

.. list-table:: SMS
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``sms.recipients``
     - string
     - Comma-separated phone numbers
   * - ``sms.text``
     - string
     - Message with placeholders

.. list-table:: Webhook
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``webhook.url``
     - string
     - Target URL with placeholders
   * - ``webhook.method``
     - string
     - HTTP method: GET, POST, PUT, DELETE
   * - ``webhook.body``
     - string
     - Request body with placeholders

.. list-table:: App
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``app.appId``
     - string
     - Target application ID

.. list-table:: I/O Signal
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``iosignal.unit``
     - string
     - Target I/O unit UUID
   * - ``iosignal.signal``
     - string
     - Target I/O signal UUID

.. list-table:: MQTT
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``mqtt.brokerAddress``
     - string
     - MQTT broker hostname
   * - ``mqtt.brokerPort``
     - int
     - MQTT broker port (default 1883)
   * - ``mqtt.topic``
     - string
     - Topic with placeholders
   * - ``mqtt.data``
     - string
     - Payload with placeholders

.. list-table:: VictoriaMetrics
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``vmetrics.metricName``
     - string
     - Metric name with placeholders

Placeholders available in all destination text fields:

.. list-table:: Placeholders
   :header-rows: 1

   * - Placeholder
     - Description
   * - ``{{NAME}}``
     - Alert signal name
   * - ``{{STATE}}``
     - Current state ("OK" or "ALARM")
   * - ``{{VALUE}}``
     - State as numeric (0=OK, 1=ALARM)
   * - ``{{SEVERITY}}``
     - Severity name (none/low/medium/high)
   * - ``{{CATEGORY}}``
     - Alert category
   * - ``{{CLASSIFICATION}}``
     - Severity classification (unknown/info/warning/critical)
   * - ``{{ID}}``
     - Alert signal UUID
   * - ``{{SOURCENAME}}``
     - Source I/O signal name
   * - ``{{SOURCEVALUE}}``
     - Current source measurement value
   * - ``{{SOURCELOCATION}}``
     - Source I/O unit location
   * - ``{{HOSTNAME}}``
     - Device hostname
   * - ``{{DEVICENAME}}``
     - Device description name
   * - ``{{DEVICELOCATION}}``
     - Device location

Alert Rule Schema
~~~~~~~~~~~~~~~~~

.. list-table:: Fields
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``uuid``
     - string
     - Auto
     - Unique identifier
   * - ``name``
     - string
     - Yes
     - Rule name
   * - ``disabled``
     - bool
     - No
     - Rule disabled (default: false)
   * - ``alertSignals``
     - array
     - Yes
     - Array of ``{"uuid": "<signalUuid>"}`` or ``[{"uuid": "*"}]`` for all
   * - ``destinations``
     - array
     - Yes
     - Array of ``{"uuid": "<destinationUuid>"}``
   * - ``triggers.ok``
     - bool
     - No
     - Trigger on OK state (default: false)
   * - ``triggers.alarm``
     - bool
     - No
     - Trigger on ALARM state (default: true)
   * - ``severityLevels.all``
     - bool
     - No
     - Match all severity levels
   * - ``severityLevels.none``
     - bool
     - No
     - Match "none" severity
   * - ``severityLevels.low``
     - bool
     - No
     - Match "low" severity
   * - ``severityLevels.medium``
     - bool
     - No
     - Match "medium" severity
   * - ``severityLevels.high``
     - bool
     - No
     - Match "high" severity
   * - ``repetition``
     - int
     - No
     - Repeat interval in minutes (0 or -1 = no repeat)

.. code-block:: json
   :caption: Example alert rule

   {
       "name": "Critical Temperature Alerts",
       "alertSignals": [{"uuid": "sig-uuid-1"}],
       "destinations": [
           {"uuid": "dest-email-1"},
           {"uuid": "dest-sms-1"}
       ],
       "triggers": {"ok": false, "alarm": true},
       "severityLevels": {
           "all": false,
           "none": false,
           "low": false,
           "medium": true,
           "high": true
       },
       "repetition": 30
   }


Error Handling
--------------

Standard Error Responses
------------------------

.. list-table:: HTTP status codes
   :header-rows: 1

   * - Status Code
     - Meaning
     - Typical Cause
   * - ``400 Bad Request``
     - Malformed request
     - Body too large, missing/invalid fields, invalid parameters
   * - ``401 Unauthorized``
     - No or invalid credentials
     - Missing token, expired token, invalid refresh token
   * - ``403 Forbidden``
     - Insufficient permissions
     - Token valid but role is not SystemAdministrator
   * - ``404 Not Found``
     - Resource does not exist
     - Invalid UUID, unknown endpoint
   * - ``406 Not Acceptable``
     - Requested action not possible
     - Invalid/expired license import
   * - ``409 Conflict``
     - Resource conflict
     - Duplicate login name, duplicate license ID
   * - ``422 Unprocessable Entity``
     - Unparseable content
     - Malformed license file
   * - ``500 Internal Server Error``
     - Server-side failure
     - I/O errors, service failures

Response Format
---------------

**Successful responses:**

- ``200 OK`` – body contains JSON, plain text, or binary data depending
  on the endpoint.
- ``202 Accepted`` – operation accepted but not yet complete (e.g.,
  time-series label fetching in progress).

**Error responses:**

- Body is typically an empty string ``""`` or a short plain-text error
  message.
- No standard JSON error envelope is used.

.. code-block:: http
   :caption: Example error responses

   HTTP/1.1 400 Bad Request
   Content-Length: 27

   upload file size too small

   HTTP/1.1 401 Unauthorized
   Content-Length: 25

   Authentication required

   HTTP/1.1 403 Forbidden
   Content-Length: 25

   Admin privileges required

   HTTP/1.1 404 Not Found
   Content-Length: 0


Caveats
-------

- The ``password`` field is never exposed in user GET responses.
- All UUIDs are 32-character lowercase hex strings without dashes.
- The ``hubadmin`` user account cannot be deleted.
- Time-series labels are fetched asynchronously; the first request for a
  new time range returns ``202 Accepted`` and subsequent requests return
  data once fetching completes.
- The CSV export endpoint uses chunked transfer encoding and streams
  data in real time.
- The system update upload is chunked; the server automatically initiates
  installation when the total received byte count matches the declared
  size.
- Configuration writes are persisted to a local SQLite database.
