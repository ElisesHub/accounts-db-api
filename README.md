
# Accounts API (Part Three)

## Overview

Accounts API is the database-facing API for the accounts system. It exposes account-related HTTP endpoints and retrieves account data from a MySQL database.

This service sits behind the Accounts Application API and is responsible for direct database access.

The frontend and Accounts Application API do not connect directly to MySQL. Database access is isolated in this API.

## Architecture

```text
Portfolio Frontend
        ↓ HTTP + API Key
Accounts Application API
        ↓ HTTP + API Key
Accounts API
        ↓ MySQL connection
MySQL Database
````

## Responsibilities

This project is responsible for:

* Exposing account-related HTTP endpoints to the Accounts Application API
* Authenticating requests using API key authentication
* Connecting to the MySQL accounts database
* Executing account-related database operations
* Returning account data to the Accounts Application API (and any subsequent APIs which may use it)
* Handling validation errors in a consistent response format
* Handling unexpected exceptions through a global exception handler
* Providing a database-aware health check endpoint

## Project Structure

```text
AccountsAPI
├── Application
├── Domain
├── Infrastructure
└── Presentation
```

Typical responsibilities:

* `Domain` — core account-related domain models and business rules
* `Application` — service interfaces, application services, and use-case orchestration
* `Infrastructure` — MySQL repositories, database access, API key validation, and configuration
* `Presentation` — API controllers, authentication setup, authorization policies, error responses, exception handling, and request/response models

## Technologies

* .NET / ASP.NET Core Web API
* Clean Architecture
* Domain-Driven Design
* MySQL
* MySqlConnector
* API key authentication
* Swagger / OpenAPI
* Health checks
* Docker / Docker Compose

## Authentication

The Accounts API uses API key authentication.

Requests must include the API key in the following HTTP header:

```text
x-api-key
```

All mapped controllers require the `RequireApiKey` authorization policy.

```csharp
app.MapControllers().RequireAuthorization("RequireApiKey");
```

If the API key is missing or invalid, the request is rejected before reaching the controller action.

The health check endpoint does not require authentication so Docker can check container health without an API key.

```http
GET /health
```

## Configuration and Secrets

This repository does not contain runtime secrets, API keys, or database connection strings.

For local development, sensitive configuration values are managed with .NET user secrets. These values are stored outside the repository and are not committed to Git.

When the application is run as part of the full accounts system, secret values are provided by a separate deployment repository through Docker Compose.

The application also supports container-mounted secrets through `/run/secrets` when they are provided by the runtime environment. This is optional and mainly intended for containerized deployments.

## Required Local User Secrets

The following .NET user secrets are required for local development:

```text
ConnectionStrings:AccountsDb=
AccountsApiKey=
```

Initialize user secrets from the Accounts API project directory:

```bash
dotnet user-secrets init
```

Set the required values:

```bash
dotnet user-secrets set "ConnectionStrings:AccountsDb" "Server=server;Database=dbname;Uid=userName;Pwd=yourpassword;"
dotnet user-secrets set "AccountsApiKey" "your-accounts-api-key"
```

Example local connection string:

```text
Server=localhost;Port=3306;Database=AccountsDb;Uid=accounts_user;Pwd=yourpassword;
```

Do not commit real API keys, database usernames, passwords, or database connection strings to source control.

## API Key Configuration

The Accounts API requires one API key value:

```text
AccountsApiKey
```

`AccountsApiKey` is used to validate incoming requests from the Accounts Application API.

If this value is missing, the application fails during startup.

## Database Configuration

The API connects to MySQL using the `AccountsDb` connection string:

```text
ConnectionStrings:AccountsDb
```

The connection string is loaded from .NET user secrets during local development.

The application reads the connection string using:

```csharp
builder.Configuration.GetConnectionString("AccountsDb")
```

The connection string is registered with MySqlConnector:

```csharp
builder.Services.AddMySqlDataSource(accountsDbConnectionString);
```

The same connection string is also used by the health check:

```csharp
builder.Services.AddHealthChecks()
    .AddMySql(accountsDbConnectionString);
```

The API fails during startup if `ConnectionStrings:AccountsDb` is missing.

## Database Requirements

The MySQL database must exist and contain the schema expected by the API.

Expected database:

```text
AccountsDb
```

The database should be initialized by the deployment setup using the required SQL seed or dump file.

The API may depend on stored procedures, tables, and seed data being present in the database. If the database is not initialized correctly, account endpoints may fail even if the API starts successfully.

## API Endpoints

### Health Check

```http
GET /health
```

Returns the health status of the service.

This endpoint checks whether the API can connect to the Accounts database.

It is intentionally unauthenticated so Docker can use it for container health checks.

### Accounts

The service exposes account-related endpoints through its controllers.

Expected account endpoint:

```http
GET /api/accounts
```

This endpoint retrieves account data from the MySQL database through the repository layer.

## Request Flow

A typical request follows this flow:

```text
Accounts Application API
  ↓
x-api-key validation
  ↓
Accounts API Controller
  ↓
IAccountsService
  ↓
IAccountsRepository
  ↓
MySQL Database
```

The application registers the account service:

```csharp
builder.Services.AddScoped<IAccountsService, AccountsService>();
```

It also registers the account repository:

```csharp
builder.Services.AddScoped<IAccountsRepository, AccountsRepository>();
```

This keeps controller logic separated from application logic and database access.

## Error Handling

The application uses a global exception handler:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```

Unexpected errors are handled centrally and returned using a consistent API error response format.

## Validation Errors

Invalid model state responses are customized.

Validation errors return a structured response containing:

```text
Code
Message
FieldErrors
TraceId
```

Example validation error shape:

```json
{
  "code": "ValidationError",
  "message": "One or more validation errors occurred.",
  "fieldErrors": {
    "fieldName": [
      "Validation error message"
    ]
  },
  "traceId": "request-trace-id"
}
```

## Swagger

Swagger is enabled only in development environments.

When running in development, Swagger UI is available through the configured Swagger endpoint.

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Swagger is not enabled in non-development environments.

## Running Locally

Restore dependencies:

```bash
dotnet restore
```

Configure the required local user secrets:

```bash
dotnet user-secrets set "ConnectionStrings:AccountsDb" "Server=localhost;Port=3306;Database=AccountsDb;Uid=accounts_user;Pwd=yourpassword;"
dotnet user-secrets set "AccountsApiKey" "your-accounts-api-key"
```

Run the API project:

```bash
dotnet run
```

If running from the solution root, provide the project path:

```bash
dotnet run --project path/to/AccountsAPI
```

The MySQL database must also be running and reachable through the configured `ConnectionStrings:AccountsDb` value.

## Running with Docker Compose

This project is designed to be run as part of the full accounts system through a separate deployment repository.

The deployment repository contains the `docker-compose.yaml` file used to start the Frontend, the Accounts Application API, the Accounts API, and the MySQL database together.

From the deployment repository, run:

```bash
docker compose up
```

The deployment repository is responsible for providing runtime configuration such as API keys, database connection strings, environment variables, Docker secrets, and database initialization files.

For this service, Docker Compose provides the MySQL connection string as an environment variable:

```yaml
environment:
  ConnectionStrings__AccountsDb: "Server=accountsdb;Port=3306;Database=AccountsDb;User ID=accounts_user;Password=${MYSQL_PASSWORD};"
```

In ASP.NET Core configuration, double underscores are converted into nested configuration keys. Therefore:

```text
ConnectionStrings__AccountsDb
```

is read by the application as:

```text
ConnectionStrings:AccountsDb
```

The API then reads this value using:

```csharp
builder.Configuration.GetConnectionString("AccountsDb")
```

The database password is not hard-coded in the Compose file. It is provided through the `MYSQL_PASSWORD` environment variable:

```yaml
Password=${MYSQL_PASSWORD}
```

The same password is also used by the MySQL container when creating the database user:

```yaml
environment:
  MYSQL_DATABASE: AccountsDb
  MYSQL_USER: accounts_user
  MYSQL_PASSWORD: ${MYSQL_PASSWORD:?MYSQL_PASSWORD is required}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?MYSQL_ROOT_PASSWORD is required}
```

The `:?` syntax means Docker Compose will fail to start if the required environment variable is missing.

The Accounts API also receives its API key through a Docker secret:

```yaml
secrets:
  - source: AccountsApiKey
    target: AccountsApiKey
```

The secret is defined in the deployment repository as:

```yaml
secrets:
  AccountsApiKey:
    file: ./secrets/AccountsApiKey.txt
```

At runtime, this secret is mounted into the container and loaded by the application through the optional `/run/secrets` configuration provider.

The MySQL database is initialized using the SQL dump file mounted into the MySQL container:

```yaml
volumes:
  - ./mySQLDb/accountsdb_dump_v1.sql:/docker-entrypoint-initdb.d/accountsdb_dump_v1.sql:ro
```

This file is executed only when the MySQL data volume is created for the first time.

The MySQL data is persisted in the named volume:

```yaml
volumes:
  mysql-data:
```

If the SQL dump changes and the database needs to be recreated in a local development environment, remove the volume and start the system again:

```bash
docker compose down -v
docker compose up
```

This repository contains the Accounts API source code only. Runtime orchestration, service wiring, environment variables, secrets, and database initialization are managed outside this repository.

## Configuration Validation

The application validates required configuration at startup.

Startup fails if:

* `AccountsApiKey` is missing
* `ConnectionStrings:AccountsDb` is missing

The health check fails if:

* The MySQL database is unavailable
* The connection string is incorrect
* The configured database user does not have access
* The database container is still starting
* The database has not been initialized correctly

## Troubleshooting

### The API fails on startup

Check that all required configuration values are present:

```text
ConnectionStrings:AccountsDb
AccountsApiKey
```

Also check that the connection string is valid.

### Requests return unauthorized

Check that the request includes the required API key header:

```text
x-api-key
```

Also check that the provided key matches the configured `AccountsApiKey`.

### The health check fails

Check that:

* The MySQL container is running
* The database host in the connection string is correct
* The database port is correct
* The database name is correct
* The username and password are correct
* The database user has access to the configured database
* The database has finished initializing

### Account endpoints fail

Check that:

* The MySQL database is running
* The API can connect to the database
* The required tables exist
* The required stored procedures exist
* The database seed or dump file ran successfully
* The configured database user has permission to execute the required queries or stored procedures

If the API returns an error such as:

```text
PROCEDURE AccountsDb.GetAccounts does not exist
```

the database was not initialized with the expected stored procedure.

In that case, check the database seed or dump file in the deployment repository and recreate the database volume if needed.

For local Docker Compose environments, MySQL initialization scripts only run when the database volume is created for the first time. If the seed file changes, the database volume may need to be recreated:

```bash
docker compose down -v
docker compose up
```

## Security Notes

Secrets are not stored in this repository.

Do not commit:

* API keys
* Database passwords
* Database connection strings containing credentials
* Environment-specific credentials
* Production configuration values
* Local user-secrets files
* Generated secret files

For local development, use .NET user secrets.

For containerized execution, required secret values are injected by the deployment setup through Docker Compose.

The database connection string contains sensitive credentials and should be treated as a secret.

## Disclaimer

This project is a simple prototype created for demonstration purposes only. It is provided "as is", without warranty of any kind.

The author is not responsible for any issues that may result from the use, modification, deployment, or distribution of this project, including data loss, security issues, or service interruptions.

This project is not intended to be used as-is in a production environment. Before any public or commercial deployment, review the security configuration, secrets management, database configuration, authentication flow, error handling, logs, and infrastructure settings.