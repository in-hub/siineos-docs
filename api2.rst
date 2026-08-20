Alternate API documentation
===========================

.. _authentication:

Authentication & Authorization
----------------------------

The API protects all resources except the *ping* endpoint by JWT based authentication.

| Header | Description | Example |
|--------|-------------|---------|
| ``Authorization: Bearer <token>`` | Standard bearer token. | ``Authorization: Bearer eyJhbGciOiJFZERTQSJ9...`` |
| ``x-access-token: <token>`` | Alternative header name. | ``x-access-token: eyJhbGciOiJFZERTQSJ9...`` |

The token is issued by the *SessionManager* on successful login. It contains the following claims:

```
{
  sub: "<loginName>",
  iat: "<issuedAt>",          // epoch timestamp
  exp: "<expiresAt>",          // 3 min after ``iat``
  sid: "<sessionUUID>",
  roles: ["SystemAdministrator"]   // can contain multiple roles
}
```

All protected routes require at least one of the following:

* **System administrator** – all routes that modify or read system state (e.g. user management, device configuration, firewall rules, etc.).
* **Authenticated user** – some routes (e.g. *ping*) do not require any role; they are publicly accessible.

If the token is missing or invalid the server will respond with **401 Unauthorized**.  
If the token is valid but does not contain the required role the server will respond with **403 Forbidden**.

The login flow is:

* **POST** ``/api/v1/auth/login``

  *Body* – JSON: ``{"username":"<name>","password":"<pw>"}``
  *Response* – ``200 OK`` with JSON:

  .. code-block:: json

     {
       "accessToken":"<jwt>",
       "tokenType":"Bearer",
       "refreshToken":"<uuid>"
     }

* **POST** ``/api/v1/auth/refresh`` – same body containing the JSON key ``"refreshToken"``. Returns a new access token.

* **POST** ``/api/v1/auth/logout`` – same body containing the JSON key ``"refreshToken"``. Invalidates the session.

* **DELETE** ``/api/v1/sessions/<sessionId>`` – invalidates the session identified by ``<sessionId>``. Requires a valid access token and that the token’s ``sid`` matches ``<sessionId>``.

All other routes either use the ``Authorization`` header or ``x-access-token`` header in the same manner. The token must be a valid EdDSA signed JWT as described above.

-----------------------------------------------------------------------

.. _api_design:

API Design Patterns
-------------------

* **Collections** – All manager components expose a base path that represents a collection of resources.  
  * ``GET /api/v1/<manager>/`` returns a JSON array of objects.  
  * ``POST /api/v1/<manager>/`` creates a new resource and returns its UUID.  
  * ``PUT /api/v1/<manager>/<uuid>/`` updates an existing resource.  
  * ``DELETE /api/v1/<manager>/<uuid>/`` removes a resource.

* **Individual resources** – When a request contains an identifier (``<uuid>``) the server will attempt to resolve the resource and return a single JSON object.

* **Configuration objects** – The *HttpConfigObjectRestResource* and its authenticated variant expose *ConfigurationObject* instances. Their payloads are *nested* JSON objects where each key’s value is wrapped in a ``{ "data": … }`` structure. The same pattern is used for *SignalProcessingChain* settings and for unit / signal configuration.  
  Example:

  .. code-block:: json

     {
       "preProcessingEnabled": {"data":true},
       "preProcessingExpression": {"data":"x*2"},
       "linearScalingEnabled": {"data":false},
       ...
     }

* **Arrays of resources** – *HttpRegArrayRestResource* and *HttpStoredArrayRestResource* use ordinary JSON arrays. The body of a ``POST`` request for creation is a JSON object, and a ``PUT`` request replaces the existing object.

* **Pagination** – The current implementation does not expose explicit pagination. Clients may filter by query parameters (e.g. ``?start=<ts>&end=<ts>`` for timeseries) or by `?limit=` if the endpoint supports it. The API returns the complete data set for the requested parameters.

-----------------------------------------------------------------------

.. _user_manager:

UserManager
-----------

| Method | Path | Description | Auth | Request | Response |
|--------|------|-------------|------|---------|----------|
| **GET** | ``/api/v1/users`` | Retrieve all users | *SystemAdministrator* | None | ``200 OK``: JSON array of users (password omitted) |
| **POST** | ``/api/v1/users`` | Create a new user | *SystemAdministrator* | JSON: ``{"loginName":"foo","fullName":"Foo Bar","password":"secret","role":1}`` | ``200 OK``: UUID |
| **PUT** | ``/api/v1/users/<uuid>`` | Update user details | *SystemAdministrator* | JSON: same as POST; password may be omitted to keep existing | ``200 OK`` |
| **DELETE** | ``/api/v1/users/<uuid>`` | Delete a user | *SystemAdministrator* | None | ``200 OK`` |
| **GET** | ``/api/v1/users/<uuid>/`` | Retrieve a single user | *SystemAdministrator* | None | ``200 OK`` |

*Password* is always stored as SHA‑256 hash and never returned.

-----------------------------------------------------------------------

.. _time_series_database_manager:

TimeSeriesDatabaseManager
-------------------------

| Method | Path | Description | Auth | Request | Response |
|--------|------|-------------|------|---------|----------|
| **GET** | ``/api/v1/vmetrics/timeseries`` | Query available time‑series names between ``start`` and ``end`` timestamps (milliseconds). | *SystemAdministrator* | ``?start=<ts>&end=<ts>`` | ``200 OK`` – array of series objects: `{ "name":"foo{unit_id=\"bar\"}", "hash":"<hash>" }` |
| **GET** | ``/api/v1/vmetrics/export/csv`` | Export selected series as CSV. | *SystemAdministrator* | ``?start=<ts>&end=<ts>&step=<s>&series=<id1,id2,..>&rollup=<min|max|avg|sum|cnt>&decimalSeparator=,&dateTimeFormat=&dateTimeLocale=`` | ``200 OK`` – chunked CSV stream; headers: ``Timestamp;Series1;Series2…`` |
| **POST** | ``/api/v1/vmetrics/timeseries/delete`` | Delete one or more series. | *SystemAdministrator* | JSON body: `[ "<hash1>", "<hash2>" ]` | ``200 OK`` |
| **POST** | ``/api/v1/vmetrics/reset`` | Reset the local VictoriaMetrics instance. | *SystemAdministrator* | None | ``200 OK`` |
| **POST** | ``/api/v1/vmetrics/submit`` | Internal helper used by I/O signals to push values. | None | Form URL encoded body: ``name=<series>&value=<val>&timestamp=<ts>`` | ``200 OK`` |

* ``submit`` is not exposed as a public route; it is invoked by the *SignalProcessingChain* when a signal is sampled.

-----------------------------------------------------------------------

.. _system_update_manager:

SystemUpdateManager
-------------------

| Method | Path | Description | Auth | Request | Response |
|--------|------|-------------|------|---------|----------|
| **POST** | ``/api/v1/system/update/upload/start`` | Begin an update file upload. | *SystemAdministrator* | JSON body: ``{"bytesToReceive":123456}`` | ``200 OK`` – returns bytes already received (0) |
| **POST** | ``/api/v1/system/update/upload/chunk`` | Upload a 4 MiB chunk of the update file. | *SystemAdministrator* | Binary body | ``200 OK`` – bytes already received |
| **GET** | ``/api/v1/system/update/install/state`` | Get installation progress. | *SystemAdministrator* | None | ``200 OK`` – ``{"message":"<msg>","progress":<percentage>}`` |
| **GET** | ``/api/v1/system/update/error`` | Get last error data. | *SystemAdministrator* | None | ``200 OK`` – error data string |

-----------------------------------------------------------------------

.. _system_manager:

SystemManager
-------------

| Method | Path | Description | Auth | Request | Response |
|--------|------|-------------|------|---------|----------|
| **GET** | ``/api/v1/system/information`` | Retrieve basic device information (IP addresses, OS, etc.). | *SystemAdministrator* | None | ``200 OK`` – JSON object |
| **GET** | ``/api/v1/system/metrics`` | Retrieve runtime metrics (CPU, memory, I/O). | *SystemAdministrator* | None | ``200 OK`` – JSON |
| **GET** | ``/api/v1/system/processes`` | List system processes. | *SystemAdministrator* | None | ``200 OK`` – JSON array |
| **GET** | ``/api/v1/system/networking/available-wireless-networks`` | List Wi‑Fi networks. | *SystemAdministrator* | None | ``200 OK`` – JSON |
| **GET** | ``/api/v1/system/clock`` | Get the system clock value. | *SystemAdministrator* | None | ``200 OK`` – number |
| **POST** | ``/api/v1/system/clock`` | Set the system clock value. | *SystemAdministrator* | Raw number body | ``200 OK`` |
| **POST** | ``/api/v1/system/hardware-clocks`` | Sync hardware clocks. | *SystemAdministrator* | None | ``200 OK`` |
| **GET** | ``/api/v1/storage/usage`` | Storage usage. | *SystemAdministrator* | None | ``200 OK`` – JSON |
| **POST** | ``/api/v1/storage/docker/cleanup`` | Remove all Docker files except volumes. | *SystemAdministrator* | None | ``200 OK`` |
| **DELETE** | ``/api/v1/storage/docker/volumes/<name>`` | Remove a Docker volume. | *SystemAdministrator* | None | ``200 OK`` |
| **POST** | ``/api/v1/storage/docker/reset`` | Delete all Docker files. | *SystemAdministrator* | None | ``200 OK`` |
| **GET** | ``/api/v1/system/journal/recent`` | Get recent journal messages. | *SystemAdministrator* | None | ``200 OK`` – plain text |
| **POST** | ``/api/v1/system/mail/test`` | Send a test e‑mail. | *SystemAdministrator* | JSON: ``{"recipient":"user@example.com","subject":"test","body":"Hello"}`` | ``200 OK`` |
| **POST** | ``/api/v1/system/sms`` | Send SMS. | *SystemAdministrator* | JSON: ``{"recipientNumbers":["+12345678"],"text":"Hello"}`` | ``200 OK`` |
| **POST** | ``/api/v1/system/device/identify`` | Blink LEDs to identify device. | *SystemAdministrator* | None | ``200 OK`` |
| **POST** | ``/api/v1/system/reboot`` | Reboot the device. | *SystemAdministrator* | None | ``200 OK`` |

Configuration objects are exposed under the path ``/api/v1/system/config/<id>``.  
Each configuration item is represented as a nested JSON object with a ``data`` key.  
Example:  

.. code-block:: json

   {
     "location": {"data":"Building 1, Room 234"},
     "hostname": {"data":"hub-gm"},
     "tlsCaCertificate": {"data":"<base64>"}
   }

-----------------------------------------------------------------------

.. _session_manager:

SessionManager
--------------

| Method | Path | Description | Auth | Request | Response |
|--------|------|-------------|------|---------|----------|
| **POST** | ``/api/v1/auth/login`` | Create a new session. | None | JSON body: ``{"username":"<name>","password":"<pw>"}`` | ``200 OK`` – JSON with ``accessToken``, ``refreshToken`` |
| **POST** | ``/api/v1/auth/refresh`` | Refresh the access token. | None | JSON body: ``{"refreshToken":"<uuid>"}`` | ``200 OK`` – new ``accessToken`` |
| **POST** | ``/api/v1/auth/logout`` | Invalidate a session. | None | JSON body: ``{"refreshToken":"<uuid>"}`` | ``200 OK`` |
| **DELETE** | ``/api/v1/sessions/<sessionId>`` | Close a session. | *SystemAdministrator* | None | ``200 OK`` |

-----------------------------------------------------------------------

.. _io_manager:

I/O Manager
-----------

The *IoManager* exposes the following collection paths. All endpoints are **system‑admin only**.

* **Unit management**

  | Method | Path | Body | Response |
  |--------|------|------|----------|
  | **POST** | ``/api/v1/io/units`` | Raw string: provider identifier (e.g. ``HubGM200IoProvider``). | ``200 OK`` – new unit UUID |
  | **DELETE** | ``/api/v1/io/units/<uuid>`` | None | ``200 OK`` |
  | **GET** | ``/api/v1/io/units/<uuid>/config`` | None | ``200 OK`` – unit config JSON (see *Configuration format* below) |
  | **PUT** | ``/api/v1/io/units/<uuid>/config`` | Full config JSON (see *Configuration format*) | ``200 OK`` |
  | **GET** | ``/api/v1/io/units/<uuid>/icon`` | None | Binary PNG (``content‑type: image/png``) |
  | **PUT** | ``/api/v1/io/units/<uuid>/icon`` | PNG binary | ``200 OK`` |
  | **DELETE** | ``/api/v1/io/units/<uuid>/icon`` | None | ``200 OK`` |
  | **PUT** | ``/api/v1/io/units/<uuid>/upstream`` | JSON: ``{"unit":"<uuid>","port":"<uuid>"}`` | ``200 OK`` |
  | **POST** | ``/api/v1/io/units/<uuid>/control/<action>`` | Raw body – action specific | ``200 OK`` |

* **Signal management**

  | Method | Path | Body | Response |
  |--------|------|------|----------|
  | **POST** | ``/api/v1/io/units/<uuid>/signals`` | None | ``200 OK`` – new signal UUID |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/<sigUuid>/config`` | Full signal config JSON (see *Signal format*) | ``200 OK`` |
  | **GET** | ``/api/v1/io/units/<uuid>/signals/<sigUuid>/config`` | None | ``200 OK`` – signal config JSON |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/config`` | JSON body: object mapping signal UUID → config | ``200 OK`` |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/<sigUuid>/control/<action>`` | Raw body – action specific | ``200 OK`` |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/<sigUuid>/duplicate`` | JSON body: ``{"name":"<new name>"}`` | ``200 OK`` – new UUID |
  | **DELETE** | ``/api/v1/io/units/<uuid>/signals`` | Query: ``uuids=<id1,id2,..>`` | ``200 OK`` |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/clear`` | None | ``200 OK`` |
  | **PUT** | ``/api/v1/io/units/<uuid>/signals/reset`` | JSON body: `["<uuid>","<uuid>"]` | ``200 OK`` |

* **Port management** – Each unit may expose ports.  
  * ``GET /api/v1/io/units/<unitUuid>/ports/<portUuid>/config`` – returns port config JSON.  
  * ``PUT /api/v1/io/units/<unitUuid>/ports/<portUuid>/config`` – updates port config.

* **Connections** – I/O connections are stored in the *connections* configuration array.  
  * ``GET /api/v1/io/connections`` – list all connections.  
  * ``POST /api/v1/io/connections`` – create a new connection.  
  * ``PUT /api/v1/io/connections/<uuid>`` – update an existing connection.

-----------------------------------------------------------------------

**Configuration format**

All configuration objects exposed via *HttpConfigObjectRestResource* or *AuthenticatedHttpConfigObjectRestResource* use the following JSON layout:

.. code-block:: json

   {
     "<key>": {"data": <value>},
     ...
   }

*Example – unit configuration (``/api/v1/io/units/<uuid>/config``):*

.. code-block:: json

   {
     "general": {
       "name": {"data":"My Sensor"},
       "location": {"data":"Warehouse"},
       "state": {"data":true}
     },
     "provider": {
       "providerId": {"data":"hubgm400"},
       "icon": {"data":"icon.svg"},
       "type": {"data":"Device"},
       "connectivity": {"data":"Direct"}
     },
     "signals": {
       "signal1": {
         "uuid": {"data":"<sigUuid>"},
         "name": {"data":"Temperature"},
         "enabled": {"data":true},
         "source1": {"data":{ "unit":"<uuid>", "signal":"<uuid>" }},
         "source2": {"data":null},
         "calculation": {"data":"+"}
       },
       "signal2": { ... }
     }
   }

*Signal configuration JSON (``/api/v1/io/units/<uuid>/signals/<sigUuid>/config``):*

.. code-block:: json

   {
     "name": {"data":"Temperature"},
     "uuid": {"data":"<sigUuid>"},
     "enabled": {"data":true},
     "vmetricsPushEnabled": {"data":true},
     "vmetricsPushInterval": {"data":60},
     "useCustomIdentifier": {"data":false},
     "customIdentifier": {"data":""},
     "group": {"data":""},
     "dataSeriesSet": {"data":""},
     "siPrefix": {"data":0},
     "measurementUnit": {"data":"DegreeCelsius"},
     "decimals": {"data":1},
     "minValue": {"data":0},
     "maxValue": {"data":100},
     "color": {"data":"#00dbc9"},
     "visualizationType": {"data":"gauge"},
     "measurementDataType": {"data":5},            // 5 = float
     "measurementTimestamp": {"data":0},
     "outputTimestamp": {"data":0}
   }

-----------------------------------------------------------------------

.. _synthetic_signals:

Synthetic Signals Provider
-------------------------

The *SyntheticSignalsIoProvider* offers the same REST interface as other IoProviders. The base path is ``/api/v1/io/synthetic/signals``.

* **Create** – ``POST /api/v1/io/synthetic/signals`` – body: JSON config. Response: new UUID.
* **Read** – ``GET /api/v1/io/synthetic/signals/<uuid>`` – returns full config.
* **Update** – ``PUT /api/v1/io/synthetic/signals/<uuid>`` – full JSON config.
* **Delete** – ``DELETE /api/v1/io/synthetic/signals/<uuid>`` – remove the signal.

Additional operations:

| Method | Path | Body | Response |
|--------|------|------|----------|
| **POST** | ``/api/v1/io/synthetic/signals/<src>/clone/<dst>`` | None | ``200 OK`` |
| **POST** | ``/api/v1/io/synthetic/signals/<uuid>/name/propagate`` | None | ``200 OK`` |
| **POST** | ``/api/v1/io/synthetic/signals/<uuid>/name/update`` | None | ``200 OK`` |

*All operations require system‑admin authentication.*

-----------------------------------------------------------------------

.. _licensing_manager:

LicensingManager
----------------

* **GET** | ``/api/v1/licensing/licenses`` | List all imported licenses. | *SystemAdministrator* | None | ``200 OK`` – array of license objects |
* **POST** | ``/api/v1/licensing/licenses`` | Import a license file (raw binary). | *SystemAdministrator* | License file (max 100 kB) | ``200 OK`` – license metadata JSON |
* **DELETE** | ``/api/v1/licensing/licenses/<id>`` | Remove a license. | *SystemAdministrator* | None | ``200 OK`` |

The import route does **not** accept JSON. It reads the binary payload and parses it into a :class:`LicensingCertificate` object. The server returns a JSON representation of the license metadata on success.

-----------------------------------------------------------------------

.. _alerting_manager:

AlertingManager
---------------

* **Signal configuration** – ``/api/v1/alerting/signals`` (collection of *AlertSignal*).  
  * ``GET`` – list all signals.  
  * ``POST`` – create new signal.  
  * ``PUT /api/v1/alerting/signals/<uuid>`` – update signal.  
  * ``DELETE`` – remove signal.

* **Destination configuration** – ``/api/v1/alerting/destinations`` – same CRUD pattern. Each destination type (EMail, SMS, Webhook, App, IoSignal, MQTT, VMetrics) is identified by its ``type`` property.

* **Rule configuration** – ``/api/v1/alerting/rules`` – same CRUD pattern. Rules reference signals and destinations by UUID.

* **Signal and destination objects** – Exposed via *HttpConfigObjectRestResource*; the JSON payload follows the “``data`` subkey” pattern described in *Authentication & Authorization*.

-----------------------------------------------------------------------

.. _apps_manager:

AppsManager
-----------

* **PUT** | ``/api/v1/apps/<appId>/control/<property>`` | Control an application. | *SystemAdministrator* | Body: ``true`` or ``false`` (JSON string). | ``200 OK`` |

The ``<property>`` may be one of:

* ``enabled`` – enable/disable the app.
* ``debug`` – enable/disable debug mode.
* ``trace`` – enable/disable trace mode.

The app manager publishes state changes to MQTT topics:

* ``apps/<appId>/enabled`` – ``"true"`` / ``"false"``
* ``apps/<appId>/debug`` – ``"true"`` / ``"false"``
* ``apps/<appId>/trace`` – ``"true"`` / ``"false"``

-----------------------------------------------------------------------

.. _firewall_manager:

FirewallManager
---------------

* **Rules** – Stored as arrays under the following paths:

  * ``/api/v1/firewall/rules/incoming``
  * ``/api/v1/firewall/rules/outgoing``
  * ``/api/v1/firewall/rules/ipforwarding``
  * ``/api/v1/firewall/rules/portforwarding``

  Each rule object contains:

  .. code-block:: json

     {
       "id":"<uuid>",
       "enabled":true,
       "networkProtocol":6,          // TCP (6), UDP (17)
       "action":2,                   // 1: Accept, 2: Drop
       "inputInterface":"eth0",
       "sourceAddress":"192.168.0.0/24",
       "destinationPorts":"80 443",
       "sourcePorts":"1024-65535"
     }

  * CRUD operations are provided by *HttpRegArrayRestResource* with the standard array semantics (POST to create, PUT to update, DELETE to remove, etc.).  

* **Internet sharing settings**

  | Method | Path | Body | Response |
  |--------|------|------|----------|
  | **GET** | ``/api/v1/firewall/internet/sharing`` | None | ``200 OK`` – JSON with ``enabled``, ``inputInterfaces``, ``outputInterface`` |
  | **POST** | ``/api/v1/firewall/internet/sharing`` | JSON: ``{"enabled":true,"inputInterfaces":["eth0"],"outputInterface":"eth0"}`` | ``200 OK`` |

-----------------------------------------------------------------------

.. _error_handling:

Error Handling
--------------

All error responses share the same structure:

.. code-block:: json

   {
     "error":"<error description>"
   }

Status codes:

* **400 Bad Request** – Malformed JSON, missing required fields, or body size too large. The response contains an explanatory ``error`` message.
* **401 Unauthorized** – No token or token not present. Response: ``{"error":"Authentication required"}``.
* **403 Forbidden** – Token valid but lacks required role. Response: ``{"error":"Admin privileges required"}``.
* **404 Not Found** – Resource does not exist. Response: ``{"error":"Resource not found"}``.
* **409 Conflict** – Duplicate resource (e.g. license already imported). Response: ``{"error":"Conflict"}``.
* **500 Internal Server Error** – Unexpected failure (e.g. database write failed). Response: ``{"error":"Internal server error"}``.

When an asynchronous route fails (e.g. network error to VictoriaMetrics) the server logs the error and returns a **500** with a short message. The client should retry after a back‑off period.

-----------------------------------------------------------------------

**Practical JSON Examples**

1. **Login**

   .. code-block:: json

      POST /api/v1/auth/login
      Content-Type: application/json

      {"username":"admin","password":"secret"}

      HTTP/1.1 200 OK
      {
        "accessToken":"<jwt>",
        "tokenType":"Bearer",
        "refreshToken":"<uuid>"
      }

2. **Create a user**

   .. code-block:: json

      POST /api/v1/users
      Authorization: Bearer <jwt>

      {
        "loginName":"john",
        "fullName":"John Doe",
        "password":"123456",
        "role":1          // 1 = GlobalAppAdministrator
      }

      HTTP/1.1 200 OK
      "e8c6d2b2-5a7f-4e8b-8e3b-2c4d9e5f"

3. **Update unit config**

   .. code-block:: json

      PUT /api/v1/io/units/<uuid>/config
      Authorization: Bearer <jwt>

      {
        "general": {
          "name": {"data":"Temp Sensor"},
          "location": {"data":"Warehouse 12"},
          "enabled": {"data":true}
        },
        "signals": {
          "sig1": {
            "uuid": {"data":"f5c7a1d5-2e9b-4f0a-9c2b-6d8f3a1"},
            "name": {"data":"Temperature"},
            "enabled": {"data":true},
            "source1": {"data":{"unit":"<uuid>","signal":"<uuid>"}},
            "calculation": {"data":"+"}
          }
        }
      }

      HTTP/1.1 200 OK

4. **Add a new signal**

   .. code-block:: text

      POST /api/v1/io/units/<uuid>/signals
      Authorization: Bearer <jwt>

      HTTP/1.1 200 OK
      "b4a1c3d2-0e8f-4b6c-9d2a-1f5e7b9"

5. **Retrieve timeseries**

   .. code-block:: text

      GET /api/v1/vmetrics/timeseries?start=1690828800000&end=1690832400000
      Authorization: Bearer <jwt>

      HTTP/1.1 200 OK
      [
        {"name":"temp{unit_id=\"a\",unit_type=\"sensor\"}","hash":"1a2b3c"},
        {"name":"pressure{unit_id=\"b\",unit_type=\"sensor\"}","hash":"4d5e6f"}
      ]

-----------------------------------------------------------------------

**Notes on JSON Payload Structure**

*Configuration objects* – Each key’s value is wrapped in ``{ "data": … }``.  
*Array resources* – Standard JSON arrays of objects.  
*Binary uploads* – For icons, license files, or update bundles, the request body is raw binary; the server validates size limits.

-----------------------------------------------------------------------

This documentation covers all public HTTP API endpoints, required authentication, data schemas, and typical error responses. It is designed to help developers consume the SIINEOS backend efficiently and correctly.
