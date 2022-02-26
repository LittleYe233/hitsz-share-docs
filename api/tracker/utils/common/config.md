# `config.js` - Process with project configurations

## Exported declarations of types

### `BasicConnectionConfig`

**Attributes:**

- `host` (`string`) - **(optional)** Host of a connection
- `port` (`number`) - **(optional)** Port of a connection
- `user` (`string`) - Username of a connection
- `pass` (`string`) - **(optional)** Password of a connection

### `BasicMySQLConfig`

**Attributes:**

- **(extends)** `BasicConnectionConfig`
- `db` (`string`) - Database name of a connection to MySQL server
- `tbl` (`string`) - Database table name of a connection to MySQL server

### `BasicRedisConfig`

**Attributes:**

- **(extends)** `BasicConnectionConfig`
- `db` (`number`) - **(optional)** Database index of a connection to Redis server. The default value is recommended to be set to `0`
- `key` (`string`) - Database key name of a connection to MySQL server

### `ParseConfigReturns`

**Attributes:**

- `secrets` (`Array<string>`) - **(optional)** Filenames of secret configurations
- `client` (`object`)
  - `databases` (`object`)
    - `auth_users` (`BasicMySQLConfig`) - Configurations of the MySQL database for authenticated users
    - `active_clients` (`BasicRedisConfig`) - Configurations of the Redis database for active clients
- `server` (`object`)
  - `databases` (`object`)
    - `auth_users` (`BasicMySQLConfig`) - Configurations of the MySQL database for authenticated users
    - `active_clients` (`BasicRedisConfig`) - Configurations of the Redis database for active clients

## Functions

### `parseConfig(filename, encoding)`

Parse a single configuration file.

**Parameters:**

- `filename` (`string`) - Filename of the configuration
- `encoding` (`BufferEncoding`) - **(optional)** Encoding of the configuration file. The default value is `utf8`

**Returns:** (`ParseConfigReturns`)

### `parseConfigWithSecrets(filename[, options])`

Parse a single configuration file with related secrets. It merges properties with the same name recursively. A secret configuration file is prior to what is in front of it on the `secrets` list, and all secrets are prior to the specified configuration file in the parameters.

**Parameters:**

- `filename` (`string`) - Filename of the configuration file
- `options` (`object`) - **(optional)** The default value is `{}`
  - `encoding` (`BufferEncoding`) - Encoding of all the configuration files including the secrets. The default value is `utf8`
  - `rmsecrets` (`boolean`) - Whether to remove secrets. The default value is `true`

**Returns:** (`ParseConfigReturns`)

### `parseProjectConfig([options])`

Parse the configuration of the project. The only difference from `parseConfigWithSecrets()` is that the `filename` parameter of the latter is frozen to a constant `DEFAULT_CONFIG_PATH` which is defined in this JavaScript file.

**Parameters:**

- `options` (`object`) - **(optional)** The default value is `{}`
  - `encoding` (`BufferEncoding`) - Encoding of all the configuration files including the secrets. The default value is `utf8`
  - `rmsecrets` (`boolean`) - Whether to remove secrets. The default value is `true`

**Returns:** (`ParseConfigReturns`)