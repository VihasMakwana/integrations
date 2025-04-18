# Microsoft SQL Server Integration

The Microsoft SQL Server integration package allows you to search, observe, and visualize the SQL Server audit logs, as well as performance and transaction log metrics, through Elasticsearch.

## Data streams

The Microsoft SQL Server integration collects two types of data streams: logs and metrics.

**Log** data streams provide records of events happening in Microsoft SQL Server:

* `audit`: Events from the configured Windows event log channel, providing detailed auditing information. See [SQL Server Audit](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine?view=sql-server-ver15).
* `logs`: Error logs created by the Microsoft SQL server for troubleshooting and system events.

Other log sources, such as files, are not supported.

Find more details in [Logs](#logs).

**Metrics** data streams provide insights into SQL Server performance and health:

* `performance`: Comprehensive performance counters and objects available on the server.
* `transaction_log`: Usage statistics and space utilization metrics for transaction logs.

Find more details in [Metrics](#metrics).

## Requirements

You need Elasticsearch for storing and searching your data and Kibana for visualizing and managing it.
You can use our hosted Elasticsearch Service on Elastic Cloud, which is recommended, or self-manage the Elastic Stack on your hardware.

### Microsoft SQL Server permissions

Before you can start sending data to Elastic, make sure you have the necessary Microsoft SQL Server permissions.

If you browse Microsoft Developer Network (MSDN) for the following tables, you will find a "Permissions" section that defines the permission needed for each table (for example, [the "Permissions" section on the `sys.dm_db_log_space_usage`](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-space-usage-transact-sql?view=sql-server-ver15#permissions) page).

1. `transaction_log`:
    - [sys.databases](https://learn.microsoft.com/en-us/sql/relational-databases/system-compatibility-views/sys-sysdatabases-transact-sql?view=sql-server-ver16)
    - [sys.dm_db_log_space_usage](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-space-usage-transact-sql?view=sql-server-ver16)
    - [sys.dm_db_log_stats (DB_ID)](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-stats-transact-sql?view=sql-server-ver16) (Available on SQL Server (MSSQL) 2016 (13.x) SP 2 and later)
2. `performance`:
    - [sys.dm_os_performance_counters](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-performance-counters-transact-sql?view=sql-server-ver16)

Please make sure the user has the permissions to system as well as user-defined databases. For the particular user used in the integration, the following requirements are met:

User setup options:

- Grant specific permissions as mentioned in the MSDN pages above.
- Alteratively, use `sysadmin` role (includes all required permissions): This can be configured via SQL Server Management Studio (SSMS) in `Server Roles`. Read more about joining a role in the [SQL Server documentation](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/join-a-role?view=sql-server-ver16#SSMSProcedure).

User Mappings (using SQL Server Management Studio (SSMS)):

- Open SSMS and connect to your server.
- Navigate to "Object Explorer" > "Security" > "Logins".
- Right-click the user and select "Properties".
- In the "User Mapping" tab, select the appropriate database and grant the required permissions.

## Setup

For step-by-step instructions on how to set up any integration, refer to the
{{ url "getting-started-observability" "Getting started" }} guide.

Below you'll find more specific details on setting up the Microsoft SQL Server integration.

### Named Instance

Microsoft SQL Server has a feature that allows running multiple databases on the same host (or clustered hosts) with separate settings. Establish a named instance connection by using the instance name along with the hostname (e.g. `host/instance_name` or `host:named_instance_port`) to collect metrics. Details of the host configuration are provided below.

#### Query by Instance Name or Server Name in Kibana

The data can be visualized in Kibana by filtering based on the instance name and server name. The instance name can be filtered by `mssql.metrics.instance_name` and the server name by `mssql.metrics.server_name` fields.

### Host Configuration

As part of the input configuration, you need to provide the user name, password and host details. The host configuration supports both named instances or default (no-name) instances, using the syntax below.

>Note: This integration supports collecting metrics from a single host. For multi-host metrics, each host can be run as a new integration.

**Connecting to Default Instance (host)**:

* `host`        (e.g. `localhost` (Instance name is not needed when connecting to default instance))
* `host:port`   (e.g. `localhost:1433`)

**Connecting to Named Instance (host)**:

* `host/instance_name`          (e.g. `localhost/namedinstance_01`)
* `host:named_instance_port`    (e.g. `localhost:60873`)



### Configuration

#### Audit

There are several levels of auditing for SQL Server, depending on government or standards requirements for your installation. The SQL Server Audit feature enables you to audit server-level and database-level groups of events and individual events.

For more information on the different audit levels, refer to [SQL Server Audit Action Groups and Actions](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15).
Then to enable auditing for SQL Server, refer to these [instructions](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/create-a-server-audit-and-server-audit-specification?view=sql-server-ver15).

>Note: For the integration package to be able to read and send audit events the event target must be configured to be Windows event log.

#### Audit events

Collects SQL Server audit events from the specified windows event log channel.

#### Log

The SQL Server `log` contains user-defined events and certain system events you can use for troubleshooting.

Read more in [View the SQL Server error log in SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/relational-databases/performance/view-the-sql-server-error-log-sql-server-management-studio?view=sql-server-ver16).

#### Performance metrics

Collects the `performance` counter metrics. The dynamic counter feature provides flexibility to collect metrics by providing the counter as an input.
This input can be a regular expression which will filter results based on pattern.
For example, if %grant% is given as input, it will enable metrics collection for all of the counters with names like 'Memory Grants Pending', 'Active memory grants count' etc.
MSSQL supports a limited set of regular expressions. For more details, refer to [Pattern Matching in Search Conditions](https://learn.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms187489(v=sql.105)?redirectedfrom=MSDN).

> Note: Dynamic counters will go through some basic ingest pipeline post-processing to make counter names in lowercase and remove special characters and these fields will not have any static field mappings.

The feature `merge_results` has been introduced in 8.4 beats which creates a single event by combining the metrics in a single event. For more details, refer to [SQL module](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-sql.html#_example_merge_multiple_queries_into_a_single_event).

Read more in [instructions about each performance counter metrics](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-performance-counters-transact-sql?view=sql-server-ver15).

#### Transaction log metrics

The system-level database `transaction_log` metrics for SQL Server instances are collected by default. Metrics for user-level databases can be collected by specifying a list of user databases or by enabling the `Fetch from all databases` toggle to collect metrics from all databases on the server.

Read more in [instructions and the operations supported by transaction log](https://docs.microsoft.com/en-us/sql/relational-databases/logs/the-transaction-log-sql-server?view=sql-server-ver15).

#### Fetch from all databases

To simplify the process of fetching metrics from all databases on the server, you can enable the `Fetch from all databases` toggle when configuring the integration. This field overrides manually entered database names in the `Databases` input and instead fetches the required `transaction_log` metrics from all databases, including system and user-defined databases.

Keep in mind that this feature is disabled by default and needs to be manually enabled to be activated.

#### Password URL encoding

When the password contains special characters, pass these special characters using URL encoding.

## Logs

### audit

The SQL Server audit dataset provides events from the configured Windows event log channel. All SQL Server audit-specific fields are available in the `sqlserver.audit` field group.

**ECS Field Reference**

Please refer to the following [document](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html) for detailed information on ECS fields.

{{fields "audit"}}

### log

The Microsoft SQL Server `log` dataset parses error logs created by the Microsoft SQL server.

{{event "log"}}

**ECS Field Reference**

Please refer to the following [document](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html) for detailed information on ECS fields.

{{fields "log"}}

## Metrics

### performance

The Microsoft SQL Server `performance` dataset provides metrics from the performance counter table. All `performance` metrics will be available in the `sqlserver.metrics` field group.

{{event "performance"}}

**ECS Field Reference**

Please refer to the following [document](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html) for detailed information on ECS fields.

{{fields "performance"}}

### transaction_log

The Microsoft SQL Server `transaction_log` dataset provides metrics from the log space usage and log stats tables. All `transaction_log` metrics will be available in the `sqlserver.metrics` field group.

{{event "transaction_log"}}

**ECS Field Reference**

Please refer to the following [document](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html) for detailed information on ECS fields.

{{fields "transaction_log"}}
