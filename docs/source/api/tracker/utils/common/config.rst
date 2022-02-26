``config.js`` -- Process with project configurations
====================================================

**Repository:** `hikarushare/tracker <https://github.com/hikarushare/tracker>`_

**Path:** `utils/common/config.js <https://github.com/hikarushare/tracker/blob/main/utils/common/config.js>`_

Metadata
--------

:File: config.js
:Author: LittleYe233
:Email: 30514318+LittleYe233@users.noreply.github.com
:Date created: 2022-02-13
:Brief: An utility to process with project configurations.

Exported declarations of ``type``\s
-----------------------------------

.. js:class:: BasicConnectionConfig

    **Role:** ``type`` in TypeScript

    :param string host:
        *(optional)* host of a connection
    :param number port:
        *(optional)* port of a connection
    :param string user:
        username of a connection
    :param string pass:
        *(optional)* password of a connection
    :returns object:

.. js:class:: BasicMySQLConfig

    **Role:** ``type`` in TypeScript

    :param string db:
        database name of a connection of MySQL
    :param string tbl:
        table name of a connection of MySQL
    :returns object:

    **Extends:** ``BasicConnectionConfig``

.. js:class:: BasicRedisConfig

    **Role:** ``type`` in TypeScript

    :param number db:
        *(optional)* database index of a connection of Redis
    :param string key:
        key of a connection of Redis
    :returns: object

    **Extends:** ``BasicConnectionConfig``