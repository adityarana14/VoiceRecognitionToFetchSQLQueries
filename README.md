

Query a database using natural language.

Thanks to [Adam Buckley](https://github.com/happyadam73/tsql-chatgpt) for the [original inspiration](https://www.linkedin.com/pulse/query-your-data-azure-sql-using-natural-language-chatgpt-adam-buckley/) for this project.

### Usage

Currently, SQL Server, PostgreSQL and SQLite databases are supported.

#### Using DatabaseGpt in your project

If you want to use **DatabaseGpt** as a library in your application, you can reference the `src/DatabaseGpt/DatabaseGpt.csproj` project and the one that contains the specific implementation for your DBMS, available as `src/DatabaseGpt.<DBMS>/DatabaseGpt.<DBMS>.csproj`:

Database|Project to include
-|-
SQL Server|src/DatabaseGpt.SqlServer/DatabaseGpt.SqlServer.csproj
PostgreSQL|src/DatabaseGpt.Npgsql/DatabaseGpt.Npgsql.csproj
SQLite|src/DatabaseGpt.Sqlite/DatabaseGpt.Sqlite.csproj

After referencing the proper projects, you can easily initialize **DatabaseGpt** at the startup of your application.

```csharp
// ...

builder.Services.AddDatabaseGpt(database =>
{
    // For SQL Server.
    database.UseConfiguration(context.Configuration)
            .UseSqlServer(context.Configuration.GetConnectionString("SqlConnection"));

    // For PostgreSQL.
    //database.UseConfiguration(context.Configuration)
    //        .UseNpgsql(context.Configuration.GetConnectionString("NpgsqlConnection"));

    // For SQLite.
    //database.UseConfiguration(context.Configuration)
    //        .UseSqlite(context.Configuration.GetConnectionString("SqliteConnection"));
},
chatGpt =>
{
    chatGpt.UseConfiguration(context.Configuration);
});
```

You need to set the required values in the **appsettings.json** file:

```
"ConnectionStrings": {
    "SqlConnection": ""             // The SQL Server connection string    
    //"NpgsqlConnection": ""        // The PostgreSQL connection string
    //"SqliteConnection": ""        // The SQLite connection string
},
"ChatGPT": {
    "Provider": "OpenAI",           // Optional. Allowed values: OpenAI (default) or Azure
    "ApiKey": "",                   // Required
    "Organization": "",             // Optional, used only by OpenAI
    "ResourceName": "",             // Required when using Azure OpenAI Service
    "AuthenticationType": "ApiKey", // Optional, used only by Azure OpenAI Service. Allowed values: ApiKey (default) or ActiveDirectory
    "DefaultModel": "my-model"      // Required  
},
"DatabaseGptSettings": {
    "IncludedTables": [ ],          // Array of table names to include (in the form of "schema.table")
    "ExcludedTables": [ ],          // Array of table names to exclude (in the form of "schema.table")
    "ExcludedColumns": [ ],         // Array of column names to exclude (in the form of "schema.table.column" to exclude a specific column, or "column" to exclude the column in all tables)
    "MaxRetries": 3                 // Max retries when the query fails
}
```

> **Note**
If possible, use GPT-4 models. Current experiments demonstrate that they are more accurate than GPT-3 models when generating queries.


### Configuration

The system works by using an OpenAI model to generate a SQL query from a natural language question, reading the list of the available tables with their structure. If table names and columns are well defined, the library should be able to automatically determine what tables to use and how to join them. For example:

```sql
CREATE TABLE dbo.Categories(
	Id INT IDENTITY(1,1) NOT NULL,
	CategoryName NVARCHAR(15) NOT NULL
)

CREATE TABLE dbo.Suppliers(
	Id INT IDENTITY(1,1) NOT NULL,
	CompanyName NVARCHAR(40) NOT NULL,
	ContactName NVARCHAR(30) NULL
)

CREATE TABLE dbo.Products(
	Id INT IDENTITY(1,1) NOT NULL,
	ProductName NVARCHAR(40) NOT NULL,
	SupplierId INT NULL,
	CategoryId INT NULL,
	QuantityPerUnit NVARCHAR(20) NULL,
	UnitPrice MONEY NULL,
	UnitsInStock SMALLINT NULL,
	UnitsOnOrder SMALLINT NULL,
	Discontinued BIT NOT NULL
)
```

Giving this schema, the model will be able to infer the following information, for example:

- If the user wants the name of the products, the column `ProductName` of the table `Products` must be used.
- The `SupplierId` column in the `Products` table is a foreign key to the `Id` column in the `Suppliers` table.
- The `CategoryId` column in the `Products` table is a foreign key to the `Id` column in the `Categories` table.

```
"DatabaseSettings": {
    "ExcludedTables": [ "dbo.CheckView" ],       
    "ExcludedColumns": [ "Timestamp" ]         // Exclude the Timestamp column from all tables
}
```

On the other hand, if you want to use only a particular set of tables, you can add them to the `IncludedTables` array in the [appsettings.json]

```json
"DatabaseSettings": {
    "IncludedTables": [ "Production.Product", "Sales.SalesOrderHeader" "Sales.SalesOrderDetail" ]
}
```

In some cases, some columns might contains values that have a particular meaning. For example:

```sql
CREATE TABLE dbo.Attachments(
	Id INT IDENTITY(1,1) NOT NULL,
	Name NVARCHAR(40) NOT NULL,
	Status INT NOT NULL,
	Path NVARCHAR(MAX) NOT NULL
)
```

```
- If the 'Status' column of table 'dbo.Attachments' is equals to 0, it means that the attachment has not been processed yet.
- If the 'Status' column of table 'dbo.Attachments' is equals to 1, it means that the attachment has been processed and approved.
- If the 'Status' column of table 'dbo.Attachments' is equals to 2, it means that the attachment has been rejected.
```

You can add as many indications as you need. The library will use this information to generate the query.

#### Retry strategy

```json
"DatabaseSettings": {
    "MaxRetries": 3
}
```

The retry strategy is handled using [Polly](https://github.com/App-vNext/Polly).

## Contribute

The project is constantly evolving. Contributions are welcome. Feel free to file issues and pull requests on the repo and we'll address them as we can. 
