# Personal Wealth & Security Vault (PWSV)

This is an educational project - a local information system for personal finance tracking. The first stage ("Transactional Core") has been implemented - the system core with a database, a server API, and a desktop client. Everything runs locally on a single PC, without any access to the internet. The main focus is on the cleanliness of the data model (3NF), integrity at the DBMS level (constraints, triggers), and basic security (password hashing, encryption of sensitive fields).

## What the system can do

Finance tracking:

- Creating and maintaining multiple accounts (cash, bank cards, crypto wallets, other)
- Support for various currencies: UAH, USD, EUR, PLN, GBP, BTC, ETH, USDT (with the ability to add your own)
- Entering exchange rates manually with a binding to a date
- Hierarchical directory of income and expense categories (unlimited nesting)
- Transactions of three types: income (Income), expense (Expense), transfer between accounts (Transfer, including between currencies)
- Automatic update of the account balance via DBMS triggers on any changes to transactions
- Protection against deleting an account that has transactions (`INSTEAD OF DELETE` trigger)
- Filtering and pagination of transactions by date, account, category, type
- Account details with a history of the last 50 transactions
- The initial balance when creating an account is recorded as a separate service transaction (category "Initial balance")
- The audit log (AuditEntries) automatically records INSERT/UPDATE/DELETE for the main entities

Security:

- Registration and login via username and password (one user per installation)
- Password hashing with the BCrypt algorithm (work factor = 12)
- Encryption of account numbers and transaction descriptions (AES-256-GCM)
- The encryption key is derived from the user's master password via PBKDF2-SHA256 (100,000 iterations, a unique salt per user)
- JWT authorization between the client and the API (HS256, valid for 8 hours)
- Rate limiting on the login endpoint (3 attempts / minute per IP)
- A server log with masking of sensitive fields (passwords, tokens, descriptions)
- A correlation ID on every HTTP request
- Verification of the resource's ownership by the user on every operation
- Localization in Ukrainian (uk-UA)

What is in the project:

- Server side on ASP.NET Core Web API (.NET 10, C# 14)
- Client side on WPF (MVVM, CommunityToolkit.Mvvm)
- SQL Server Express LocalDB database
- Entity Framework Core 10 for working with the DB
- Database-level triggers for automatic recalculation of balances
- Clean architecture (Domain / Application / Infrastructure / Presentation)
- CQRS via MediatR with pipeline behaviors (logging, validation, unit-of-work for transactional commands)
- FluentValidation on the Application side
- Mapster for Entity → DTO mapping
- Serilog with structured logging to files
- Swagger / OpenAPI for API documentation
- Unit tests (xUnit + FluentAssertions) and integration tests (WebApplicationFactory + real LocalDB)

## What needs to be installed

- Windows 10 (x64) or newer
- Visual Studio 2026 Community with the workloads:
  - ".NET desktop development"
  - "ASP.NET and web development"
  - "Data storage and processing" (for LocalDB)
- .NET 10 SDK (included with VS 2026)
- SQL Server Express LocalDB (usually installed together with VS)

There is no need to install any additional packages by hand - NuGet will pull everything in on the first build.

## How to run locally

### 1. Download the repository from GitHub

```
git clone https://github.com/ldksfld/PWSV.git
cd PWSV
```

### 2. Make sure LocalDB is alive

At the command line:

```
sqllocaldb info
```

The list should contain `MSSQLLocalDB`. If it is not there - create and start it:

```
sqllocaldb create "MSSQLLocalDB"
sqllocaldb start "MSSQLLocalDB"
```

Checking the status:

```
sqllocaldb info "MSSQLLocalDB"
```

The output should contain `State: Running`.

### 3. Trust the HTTPS dev certificate

Needed only if this is the first .NET project on this machine:

```
dotnet dev-certs https --trust
```

### 4. Restore NuGet packages and build the solution (Build)

From the root of the repository:

```
dotnet restore PWSV.slnx
dotnet build PWSV.slnx -c Debug
```

Or via Visual Studio:

1. Open the `PWSV.slnx` file in Visual Studio 2026.
2. Wait for the NuGet packages to be restored (this may take a minute or two).
3. `Build > Build Solution` (`Ctrl+Shift+B`). The build should pass without errors and warnings (the project is built with `TreatWarningsAsErrors=true`).

### 5. Configure the JWT secret

There is nothing to create separately - on the first run of `PWSV.Api` the secret will be generated automatically into the file `src/PWSV.Api/secrets.json` (48 random bytes, base64). This file is excluded from the repository via `.gitignore` and is used only locally.

If you want to set your own secret (for example, so that JWT tokens survive a restart, or to set a specific value across machines) - create the file `src/PWSV.Api/secrets.json` manually:

```json
{
  "Jwt": {
    "SecretKey": "a_long_random_string_here_at_least_32_bytes"
  }
}
```

Alternatively - via User Secrets (a value from User Secrets overrides `secrets.json`):

```
dotnet user-secrets init --project src/PWSV.Api
dotnet user-secrets set "Jwt:SecretKey" "a_long_random_string_here_at_least_32_bytes" --project src/PWSV.Api
```

### 6. Database

Running the migrations separately is not mandatory - on the first run of `PWSV.Api` the following will be performed:

- creation of the DB files in the directory `%LOCALAPPDATA%\PWSV\PWSV.mdf` (and `PWSV_log.ldf`),
- application of all EF Core migrations (schema, check constraints, indexes, triggers on `Transactions` and `Accounts`),
- population of the `AccountTypes` and `Currencies` directories with base records.

If you still need to run the migrations before launching (for example, from the command line), first install the EF Core CLI as a global tool (if it was not installed yet), then perform the update:

```
dotnet tool install --global dotnet-ef
dotnet ef database update --project src/PWSV.Infrastructure --startup-project src/PWSV.Api
```

Or from the `Package Manager Console` in Visual Studio (`Tools > NuGet Package Manager > Package Manager Console`):

```
Update-Database -Project PWSV.Infrastructure -StartupProject PWSV.Api
```

To verify that the triggers were actually created - via `View > SQL Server Object Explorer` in Visual Studio, connect to `(localdb)\MSSQLLocalDB`, open the `PWSV` database. The `dbo.Transactions` table should contain the triggers:

- `trg_Transactions_AfterInsert`
- `trg_Transactions_AfterUpdate`
- `trg_Transactions_AfterDelete`

And in the `dbo.Accounts` table:

- `trg_Accounts_PreventDeleteWhenHasTransactions`

If you need to reset the database completely: close Visual Studio, run:

```
sqllocaldb stop "MSSQLLocalDB"
```

delete the `%LOCALAPPDATA%\PWSV` directory completely, after which start the API again - the database will be created from scratch.

### 7. Run the API and the client

In Visual Studio:

1. Right-click the `PWSV` solution → `Configure Startup Projects`.
2. Select `Multiple startup projects`.
3. Set Action = `Start` for `PWSV.Api` and `PWSV.Client`.
4. OK, then `F5`.

Or from the command line in two separate terminals (important: for the API, explicitly specify the `https` profile, because the `http` profile by default listens only on port 5000, while the client accesses `https://localhost:5001`):

```
dotnet run --project src/PWSV.Api --launch-profile https
dotnet run --project src/PWSV.Client
```

First the API will start on `https://localhost:5001` (Swagger - at `https://localhost:5001/swagger`), then the WPF client window will open.

### 8. Registering the first user

In the client window, register a user (username + password + master password). Requirements:

- Username: 3-64 characters.
- Password: 8-128 characters, must contain a lowercase letter, an uppercase letter, a digit.
- Master password: at least 8 characters. It is used to derive the AES key and is requested on every login - if lost, the encrypted fields (account numbers, descriptions) will be impossible to decrypt.

After registration, login will happen automatically, and the main window will open. Registration is available only once - the system is designed for one user per installation.
