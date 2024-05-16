# Fineract

## Database Deployment

```
CREATE USER fin_dev WITH PASSWORD 'xxxx' CREATEDB;
CREATE DATABASE fineract_tenants;
CREATE DATABASE fineract_default;
GRANT ALL PRIVILEGES ON DATABASE fineract_tenants TO fin_dev;
GRANT ALL PRIVILEGES ON DATABASE fineract_default TO fin_dev;
```

### Multi Tenancy

Multi-tenancy Fineract 1.x supports multi-tenancy. The services are multi-tenant capable. Data is placed in separate dataresources for each tenant. Which tenant is addressed by a request is transmitted via the request header, and via the bearer token. Any deployment may serve multiple tenants.

How to create another tenant for the same instance. This can be done by:

#### Create an empty database.

Creating a new database called `fineract_test` (could be any name, but must be prefixed with `fineract_`)

```sql
create database fineract_djamo-ci;
GRANT ALL PRIVILEGES ON DATABASE fineract_djamo-ci TO fin_dev;
```

#### Add the new tenant

Executing the following sql query to add connection details for the new tenant to the `fineract_tenants.tenant_server_connections` table:

```sql
INSERT INTO `tenant_server_connections` VALUES ('localhost', '[NEW TENANT DATAASE NAME]', '3306', 'root', 'mysql', 1, 5, 30000, 1, 60, 1, 50, 1, 40, 20, 10, 60, 34000, 60000, 0, 1);
```

```sql
-- Real example

INSERT INTO public.tenant_server_connections(
	id, schema_server, schema_name, schema_server_port, schema_username, schema_password, auto_update, pool_initial_size, pool_validation_interval, pool_remove_abandoned, pool_remove_abandoned_timeout, pool_log_abandoned, pool_abandon_when_percentage_full, pool_test_on_borrow, pool_max_active, pool_min_idle, pool_max_idle, pool_suspect_timeout, pool_time_between_eviction_runs_millis, pool_min_evictable_idle_time_millis, deadlock_max_retries, deadlock_max_retry_interval, schema_connection_parameters, readonly_schema_server, readonly_schema_name, readonly_schema_server_port, readonly_schema_username, readonly_schema_password, readonly_schema_connection_parameters)
	VALUES (3, 'x.x.x.x', 'fineract_djamo-ci', '5432', 'fin_dev', 'xxx', 1, 5, 30000, 1, 60, 1, 50, 1, 40, 20, 10, 60, 34000, 60000, 0, 1, NULL, NULL, NULL, NULL, NULL, NULL, NULL);
```

```sql
-- Real example

INSERT INTO public.tenant_server_connections(
	id, schema_server, schema_name, schema_server_port, schema_username, schema_password, auto_update, pool_initial_size, pool_validation_interval, pool_remove_abandoned, pool_remove_abandoned_timeout, pool_log_abandoned, pool_abandon_when_percentage_full, pool_test_on_borrow, pool_max_active, pool_min_idle, pool_max_idle, pool_suspect_timeout, pool_time_between_eviction_runs_millis, pool_min_evictable_idle_time_millis, deadlock_max_retries, deadlock_max_retry_interval, schema_connection_parameters, readonly_schema_server, readonly_schema_name, readonly_schema_server_port, readonly_schema_username, readonly_schema_password, readonly_schema_connection_parameters)
	VALUES (4, 'x.x.x.x', 'fineract_bank-d_sandbox', '5432', 'fin_dev', 'xxx', 1, 5, 30000, 1, 60, 1, 50, 1, 40, 20, 10, 60, 34000, 60000, 0, 1, NULL, NULL, NULL, NULL, NULL, NULL, NULL);
```

Where:
“fineract_test” is the name of the new tenant database
“root” is the username used to connect to the database
“mysql” is the password used to connect to the database

Inserting into fineract_tenants.tenants table a new tenant:

```sql
--
-- Dumping data for table `tenants`
--

LOCK TABLES `tenants` WRITE;
/*!40000 ALTER TABLE `tenants` DISABLE KEYS */;
INSERT INTO `tenants` VALUES
(2,'default','default','fineract_default','Asia/Kolkata',NULL,NULL,NULL,NULL,'localhost','3306',NULL,'root','mysql',1,5,30000,1,60,1,50,1,40,20,10,60,34000,60000);
/*!40000 ALTER TABLE `tenants` ENABLE KEYS */;
UNLOCK TABLES;
```

```sql
INSERT INTO public.tenants(
	id, identifier, name, timezone_id, country_id, joined_date, created_date, lastmodified_date, oltp_id, report_id)
	VALUES (3, 'djamo-ci', 'djamo-ci', 'Africa/Abidjan', 118, NULL, NULL, NULL, 3, 3);
```

Where: `oltp_id` and `report_id` represents the new tenant connection id. Set this properly otherwise it may connect to other database.

```sql
use `mifosplatform-tenants`;

INSERT INTO `tenant_server_connections` (`schema_name`) VALUES ('mifostenant-reference');

INSERT INTO `tenants` (`identifier`, `name`, `oltp_id`, `report_id`, `timezone_id`) VALUES ('reference', 'reference', ( select id from tenant_server_connections where schema_name="mifostenant-reference"), (select id from tenant_server_connections where schema_name="mifostenant-reference"), 'Asia/Kolkata');

create database `mifostenant-reference`;

use `mifostenant-reference`;

source load_sample_data.sql ;
```

#### Restart Server

Restart the tomcat server the application is running on.

Note: Set the lock to false in the table `databasechangeloglock` if there is any database lock issue during startup.

```
Database name: fineract_test
Tenant Identifier: test
```

Community App url E.G: https://YOURDOMAIN/?baseApiUrl=YOURDOMAIN:8443/fineract-provider&tenantIdentifier=test

Login Credentials as set (default):

```
Username: mifos
Password: password
```

#### Reference

Few relevant documentation to use here:

https://research.muellners.org/fineract-common-deployment-issues/
https://docs.fineract.net/architecture/multi-tenancy
https://cwiki.apache.org/confluence/display/FINERACT/Fineract+101
https://mifosforge.jira.com/wiki/spaces/docs/pages/187498786/How+to+Setup+New+Trail+Instance
https://gist.github.com/vishwasbabu/03ed5d3bc3c2687807b9

## Application Deployment

Note: Kubernetes automatically set service address in the pod's environment variable, so do not call the service `fineract-server`. It will automatically set `FINERACT_SERVER_PORT` to the Kubernetes service address, which is conflict with the fineract variable.

### Deploy with Kustomize

```bash
kubectl apply -k ./fineract/dev
```

### Health Check

```
kubectl exec --stdin --tty fineract-server-579574d69c-ff29r -- /bin/bash
```

```
http://127.0.0.1:8080/fineract-provider/actuator/health
```
