.. Licensed to the Apache Software Foundation (ASF) under one
.. or more contributor license agreements.  See the NOTICE file
.. distributed with this work for additional information
.. regarding copyright ownership.  The ASF licenses this file
.. to you under the Apache License, Version 2.0 (the
.. "License"); you may not use this file except in compliance
.. with the License.  You may obtain a copy of the License at
..
..   http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing,
.. software distributed under the License is distributed on an
.. "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
.. KIND, either express or implied.  See the License for the
.. specific language governing permissions and limitations
.. under the License.

================
Snowflake Driver
================

**Available for:** C/C++, GLib/Ruby, Go, Python, R

The Snowflake Driver provides access to Snowflake Database Warehouses.

Installation
============

.. tab-set::

   .. tab-item:: C/C++
      :sync: cpp

      For conda-forge users:

      .. code-block:: shell

         mamba install libadbc-driver-snowflake

   .. tab-item:: Go
      :sync: go

      .. code-block:: shell

        go get github.com/apache/arrow-adbc/go/adbc/driver/snowflake

   .. tab-item:: Python
      :sync: python

      .. code-block:: shell

         # For conda-forge
         mamba install adbc-driver-snowflake

         # For pip
         pip install adbc_driver_snowflake

   .. tab-item:: R
      :sync: r

      .. code-block:: shell

         # install.packages("pak")
         pak::pak("apache/arrow-adbc/r/adbcsnowflake")

Usage
=====

To connect to a Snowflake database you can supply the "uri" parameter when
constructing the :cpp::class:`AdbcDatabase`.

.. tab-set::

   .. tab-item:: C++
      :sync: cpp

      .. code-block:: cpp

         #include "adbc.h"

         // Ignoring error handling
         struct AdbcDatabase database;
         AdbcDatabaseNew(&database, nullptr);
         AdbcDatabaseSetOption(&database, "driver", "adbc_driver_snowflake", nullptr);
         AdbcDatabaseSetOption(&database, "uri", "<snowflake uri>", nullptr);
         AdbcDatabaseInit(&database, nullptr);

   .. tab-item:: Python
      :sync: python

      .. code-block:: python

         import adbc_driver_snowflake.dbapi

         with adbc_driver_snowflake.dbapi.connect("<snowflake uri>") as conn:
             pass

   .. tab-item:: R
      :sync: r

      .. code-block:: r

         library(adbcdrivermanager)

         # Use the driver manager to connect to a database
         uri <- Sys.getenv("ADBC_SNOWFLAKE_TEST_URI")
         db <- adbc_database_init(adbcsnowflake::adbcsnowflake(), uri = uri)
         con <- adbc_connection_init(db)

   .. tab-item:: Go
      :sync: go

      .. code-block:: go

         import (
            "context"

            "github.com/apache/arrow-adbc/go/adbc"
            "github.com/apache/arrow-adbc/go/adbc/driver/snowflake"
         )

         func main() {
            var drv snowflake.Driver
            db, err := drv.NewDatabase(map[string]string{
                adbc.OptionKeyURI: "<snowflake uri>",
            })
            if err != nil {
                // handle error
            }

            cnxn, err := db.Open(context.Background())
            if err != nil {
                // handle error
            }
            defer cnxn.Close()
         }

URI Format
----------

The Snowflake URI should be of one of the following formats:

- ``user[:password]@account/database/schema[?param1=value1&paramN=valueN]``
- ``user[:password]@account/database[?param1=value1&paramN=valueN]``
- ``user[:password]@host:port/database/schema?account=user_account[&param1=value1&paramN=valueN]``
- ``host:port/database/schema?account=user_account[&param1=value1&paramN=valueN]``

Alternately, instead of providing a full URI, the configuration can
be entirely supplied using the other available options or some combination
of the URI and other options. If a URI is provided, it will be parsed first
and any explicit options provided will override anything parsed from the URI.

Supported Features
==================

The Snowflake driver generally supports features defined in the ADBC API
specification 1.0.0, as well as some additional, custom options.

Authentication
--------------

Snowflake requires some form of authentication to be enabled. By default
it will attempt to use Username/Password authentication. The username and
password can be provided in the URI or via the ``username`` and ``password``
options to the :cpp:class:`AdbcDatabase`.

Alternately, other types of authentication can be specified and customized.
See "Client Options" below.

Bulk Ingestion
--------------

Bulk ingestion is supported. The mapping from Arrow types to Snowflake types
is provided below.

Partitioned Result Sets
-----------------------

Partitioned result sets are not currently supported.

Performance
-----------

Formal benchmarking is forthcoming. Snowflake does provide an Arrow native
format for requesting results, but bulk ingestion is still currently executed
using the REST API. As described in the `Snowflake Documentation
<https://pkg.go.dev/github.com/snowflakedb/gosnowflake#hdr-Batch_Inserts_and_Binding_Parameters>`
the driver will potentially attempt to improve performance by streaming the data
(without creating files on the local machine) to a temporary stage for ingestion
if the number of values exceeds some threshold.

In order for the driver to leverage this temporary stage, the user must have
the ``CREATE STAGE`` privilege on the schema. If the user does not have this
privilege, the driver will fall back to sending the data with the query
to the snowflake database.

In addition, the current database and schema for the session must be set. If
these are not set, the ``CREATE TEMPORARY STAGE`` command executed by the driver
can fail with the following error:

.. code-block::

   CREATE TEMPORARY STAGE SYSTEM$BIND file_format=(type=csv field_optionally_enclosed_by='"')
   CANNOT perform CREATE STAGE. This session does not have a current schema. Call 'USE SCHEMA' or use a qualified name.

In addition, results are potentially fetched in parallel from multiple endpoints.
A limited number of batches are queued per endpoint, though data is always
returned to the client in the order of the endpoints.

To manage the performance of result fetching there are two options to control
buffering and concurrency behavior. These options are only available to be set
on the :cpp:class:`AdbcStatement` object:

``adbc.rpc.result_queue_size``
    The number of batches to queue in the record reader. Defaults to 200.
    Must be an integer > 0.

``adbc.snowflake.rpc.prefetch_concurrency``
    The number of concurrent streams being fetched from snowflake at a time.
    Defaults to 10. Must be an integer > 0.

Transactions
------------

Transactions are supported. Keep in mind that Snowflake transactions will
implicitly commit if any DDL statements are run, such as ``CREATE TABLE``.

Client Options
--------------

The options used for creating a Snowflake Database connection can be customized.
These options map 1:1 with the Snowflake `Config object <https://pkg.go.dev/github.com/snowflakedb/gosnowflake#Config>`.

``adbc.snowflake.sql.db``
    The database this session should default to using.

``adbc.snowflake.sql.schema``
    The schema this session should default to using.

``adbc.snowflake.sql.warehouse``
    The warehouse this session should default to using.

``adbc.snowflake.sql.role``
    The role that should be used for authentication.

``adbc.snowflake.sql.region``
    The Snowflake region to use for constructing the connection URI.

``adbc.snowflake.sql.account``
    The Snowflake account that should be used for authentication and building the
    connection URI.

``adbc.snowflake.sql.uri.protocol``
    This should be either `http` or `https`.

``adbc.snowflake.sql.uri.port``
    The port to use for constructing the URI for connection.

``adbc.snowflake.sql.uri.host``
    The explicit host to use for constructing the URL to connect to.

``adbc.snowflake.sql.auth_type``
    Allows specifying alternate types of authentication, the allowed values are:

    - ``auth_snowflake``: General username/password authentication (this is the default)
    - ``auth_oauth``: Use OAuth authentication for the snowflake connection.
    - ``auth_ext_browser``: Use an external browser to access a FED and perform SSO auth.
    - ``auth_okta``: Use a native Okta URL to perform SSO authentication using Okta
    - ``auth_jwt``: Use a provided JWT to perform authentication.
    - ``auth_mfa``: Use a username and password with MFA.

``adbc.snowflake.sql.client_option.auth_token``
    If using OAuth or another form of authentication, this option is how you can
    explicitly specify the token to be used for connection.

``adbc.snowflake.sql.client_option.okta_url``
    If using ``auth_okta``, this option is required in order to specify the
    Okta URL to connect to for SSO authentication.

``adbc.snowflake.sql.client_option.login_timeout``
    Specify login retry timeout *excluding* network roundtrip and reading http responses.
    Value should be formatted as described `here <https://pkg.go.dev/time#ParseDuration>`,
    such as ``300ms``, ``1.5s`` or ``1m30s``. Even though negative values are accepted,
    the absolute value of such a duration will be used.

``adbc.snowflake.sql.client_option.request_timeout``
    Specify request retry timeout *excluding* network roundtrip and reading http responses.
    Value should be formatted as described `here <https://pkg.go.dev/time#ParseDuration>`,
    such as ``300ms``, ``1.5s`` or ``1m30s``. Even though negative values are accepted,
    the absolute value of such a duration will be used.

``adbc.snowflake.sql.client_option.jwt_expire_timeout``
    JWT expiration will occur after this timeout.
    Value should be formatted as described `here <https://pkg.go.dev/time#ParseDuration>`,
    such as ``300ms``, ``1.5s`` or ``1m30s``. Even though negative values are accepted,
    the absolute value of such a duration will be used.

``adbc.snowflake.sql.client_option.client_timeout``
    Specify timeout for network roundtrip and reading http responses.
    Value should be formatted as described `here <https://pkg.go.dev/time#ParseDuration>`,
    such as ``300ms``, ``1.5s`` or ``1m30s``. Even though negative values are accepted,
    the absolute value of such a duration will be used.

``adbc.snowflake.sql.client_option.app_name``
    Allows specifying the Application Name to Snowflake for the connection.

``adbc.snowflake.sql.client_option.tls_skip_verify``
    Disable verification of the server's TLS certificate. Value should be ``true``
    or ``false``.

``adbc.snowflake.sql.client_option.ocsp_fail_open_mode``
    Control the fail open mode for OCSP. Default is ``true``. Value should
    be either ``true`` or ``false``.

``adbc.snowflake.sql.client_option.keep_session_alive``
    Enable the session to persist even after the connection is closed. Value
    should be either ``true`` or ``false``.

``adbc.snowflake.sql.client_option.jwt_private_key``
    Specify the RSA private key which should be used to sign the JWT for
    authentication. This should be a path to a file containing a PKCS1
    private key to be read in and parsed. Commonly encoded in PEM blocks
    of type "RSA PRIVATE KEY".

``adbc.snowflake.sql.client_option.disable_telemetry``
    The Snowflake driver allows for telemetry information which can be
    disabled by setting this to ``true``. Value should be either ``true``
    or ``false``.

``adbc.snowflake.sql.client_option.tracing``
    Set the logging level

``adbc.snowflake.sql.client_option.cache_mfa_token``
    When ``true``, the MFA token is cached in the credential manager. Defaults
    to ``true`` on Windows/OSX, ``false`` on Linux.

``adbc.snowflake.sql.client_option.store_temp_creds``
    When ``true``, the ID token is cached in the credential manager. Defaults
    to ``true`` on Windows/OSX, ``false`` on Linux.


Metadata
--------

When calling :cpp:func:`AdbcConnectionGetTableSchema`, the returned Arrow Schema
will contain metadata on each field:

``DATA_TYPE``
    This will be a string containing the raw Snowflake data type of this column

``PRIMARY_KEY``
    This will be either ``Y`` or ``N`` to indicate a column is a primary key.

In addition, the schema on the stream of results from a query will contain
the following metadata keys on each field:

``logicalType``
    The Snowflake logical type of this column. Will be one of ``fixed``,
    ``real``, ``text``, ``date``, ``variant``, ``timestamp_ltz``, ``timestamp_ntz``,
    ``timestamp_tz``, ``object``, ``array``, ``binary``, ``time``, ``boolean``.

``precision``
    An integer representing the Snowflake precision of the field.

``scale``
    An integer representing the Snowflake scale of the values in this field.

``charLength``
    If a text field, this will be equivalent to the ``VARCHAR(#)`` parameter ``#``.

``byteLength``
    Will contain the length, in bytes, of the raw data sent back from Snowflake
    regardless of the type of the field in Arrow.

Type Support
------------

Because Snowflake types do not necessary match up 1-to-1 with Arrow types
the following is what should be expected when requesting data. Any conversions
indicated are done to ensure consistency of the stream of record batches.

.. list-table::
   :header-rows: 1

   * - Snowflake Type
     - Arrow Type
     - Notes

   * - integral types
     - int64
     - All integral types in Snowflake are stored as 64-bit integers.

   * - float/double
     - float64
     - Snowflake does not distinguish between float or double. Both are 64-bit values.

   * - decimal/numeric
     - int64/float64
     - If scale == 0, then int64 is used, else float64.

   * - time
     - time64[ns]
     - For ingestion, time32 can also be used.

   * - date
     - date32
     - For ingestion, date64 can also be used.

   * - | timestamp_ltz
       | timestamp_ntz
       | timestamp_tz
     - timestamp[ns]
     - Local time zone will be used. No timezone will be specified in
       the Arrow type. Values will be converted to UTC.

   * - | variant
       | object
       | array
     - string
     - Snowflake does not provide information about nested
       types. Values will be strings in a format similar to JSON that
       can be parsed. The Arrow type will contain a metadata key
       ``logicalType`` with the Snowflake field type.

   * - | geography
       | geometry
     - string
     - There is no current canonical Arrow (extension) type for these
       types, so they will be returned as the string values that
       Snowflake provides.
