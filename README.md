# TestContainer Demo

This is a sample project that demonstrates the use of a TestContainer that provides easy and lightweight APIs for bootstrapping integration tests with real services wrapped in Docker containers. 

Using Testcontainers, you can write tests that talk to the same type of services you use in production, without mocks or in-memory services.

The following is a simple .NET class library using a Postgres database. These type of class libraries are usually implemented as a part of an infrastructure layer. 

## Setup solution and required projects

**Create a new solution**
```shell
dotnet new sln -o TestcontainersDemo
cd TestcontainersDemo
```

**Create a new class library**
```shell
dotnet new classlib -o CustomerService
dotnet sln add ./CustomerService/CustomerService.csproj
```

**Create a test project**
```shell
dotnet new xunit -o CustomerService.Tests
dotnet sln add ./CustomerService.Tests/CustomerService.Tests.csproj
dotnet add ./CustomerService.Tests/CustomerService.Tests.csproj reference ./CustomerService/CustomerService.csproj
```

**Add PostgreSQL nuget package**
```
dotnet add ./CustomerService/CustomerService.csproj package Npgsql
```

We have added Npgsql, a Postgres ADO.NET Data Provider for talking to the Postgres database, and using xUnit for testing our service.

## Add business logic


Start by createing our model `Customer` 

```csharp
namespace CustomerService.Model;

public readonly record struct Customer(long Id, string Name);
```

Then add a Database connection class:

```csharp
using System.Data.Common;
using Npgsql;

namespace CustomerService.Database;

public sealed class DbConnectionProvider
{
    private readonly string _connectionString;

    public DbConnectionProvider(string connectionString)
    {
        _connectionString = connectionString;
    }

    public DbConnection GetConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }
}
```

Finally create our customer service :

```csharp
using CustomerService.Database;
using CustomerService.Model;

namespace CustomerService;

public sealed class CustomerService
{
    private readonly DbConnectionProvider _dbConnectionProvider;

    public CustomerService(DbConnectionProvider dbConnectionProvider)
    {
        _dbConnectionProvider = dbConnectionProvider;
        CreateCustomersTable();
    }

    public IEnumerable<Customer> GetCustomers()
    {
        IList<Customer> customers = new List<Customer>();

        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();
        command.CommandText = "SELECT id, name FROM customers";
        command.Connection?.Open();

        using var dataReader = command.ExecuteReader();
        while (dataReader.Read())
        {
            var id = dataReader.GetInt64(0);
            var name = dataReader.GetString(1);
            customers.Add(new Customer(id, name));
        }

        return customers;
    }

    public void Create(Customer customer)
    {
        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();

        var id = command.CreateParameter();
        id.ParameterName = "@id";
        id.Value = customer.Id;

        var name = command.CreateParameter();
        name.ParameterName = "@name";
        name.Value = customer.Name;

        command.CommandText = "INSERT INTO customers (id, name) VALUES(@id, @name)";
        command.Parameters.Add(id);
        command.Parameters.Add(name);
        command.Connection?.Open();
        command.ExecuteNonQuery();
    }

    private void CreateCustomersTable()
    {
        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();
        command.CommandText = "CREATE TABLE IF NOT EXISTS customers (id BIGINT NOT NULL, name VARCHAR NOT NULL, PRIMARY KEY (id))";
        command.Connection?.Open();
        command.ExecuteNonQuery();
    }
}
```

The `CustomerService` class provides the following functionality:

- `_dbConnectionProvider.GetConnection()` gets a database connection using ADO.NET.

- `CreateCustomersTable()` method creates the customers table if it does not already exist.

- `GetCustomers()` method fetches all rows from the customers table, populates data into `Customer` objects, and returns a list of `Customer` objects.

- `Create(Customer)` method inserts a new `customer` record into the database.

## Use TestContainer

Before writing Testcontainers-based tests, let’s add Testcontainers PostgreSql module dependency to the test project as follows:

```shell
dotnet add ./CustomerService.Tests/CustomerService.Tests.csproj package Testcontainers.PostgreSql
```

As we are using a Postgres database for our application, we added the Testcontainers Postgres module as a test dependency.

**Write Tests**

Create a `CustomerServiceTest` class in the test project with the following code:

```csharp
using CustomerService.Database;
using CustomerService.Model;
using Testcontainers.PostgreSql;

namespace CustomerService.Tests;

public sealed class CustomerServiceTest : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:15-alpine")
        .Build();

    public Task InitializeAsync()
    {
        return _postgres.StartAsync();
    }

    public Task DisposeAsync()
    {
        return _postgres.DisposeAsync().AsTask();
    }

    [Fact]
    public void ShouldReturnTwoCustomers()
    {
        // Given
        var customerService = new CustomerService(new DbConnectionProvider(_postgres.GetConnectionString()));

        // When
        customerService.Create(new Customer(1, "George"));
        customerService.Create(new Customer(2, "John"));
        var customers = customerService.GetCustomers();

        // Then
        Assert.Equal(2, customers.Count());
    }
}
```

Let us understand the code in our `CustomerServiceTest` class.

- Declare `PostgreSqlContainer` by passing the Docker image name `postgres:15-alpine` to the Postgres builder.

- The Postgres container is started using xUnit.net’s `IAsyncLifetime` interface, which executes `InitializeAsync` immediately after the test class has been created.

- `ShouldReturnTwoCustomers()` test initializes `CustomerService`, insert two customer records into the database, fetch all the existing customers, and assert the number of customers.

- Finally, the Postgres container is disposed in the `DisposeAsync()` member, which gets executed after the test method is executed.

**Test**

Ensure that you have docker running.

Now let’s run the tests as follows:

`dotnet test`

```shell
Starting test execution, please wait...
A total of 1 test files matched the specified pattern.
[testcontainers.org 00:00:00.04] Connected to Docker:
  Host: unix:///var/run/docker.sock
  Server Version: 27.1.1
  Kernel Version: 6.10.0-linuxkit
  API Version: 1.46
  Operating System: Docker Desktop
  Total Memory: 7.66 GB
  Labels: 
    com.docker.desktop.address=unix:///Users/bengthedberg/Library/Containers/com.docker.docker/Data/docker-cli.sock

[testcontainers.org 00:00:00.11] Docker container 220c2681e3e0 created
[testcontainers.org 00:00:00.13] Start Docker container 220c2681e3e0
[testcontainers.org 00:00:00.24] Wait for Docker container 220c2681e3e0 to complete readiness checks
[testcontainers.org 00:00:00.24] Docker container 220c2681e3e0 ready
[testcontainers.org 00:00:00.27] Docker container a26577c95780 created
[testcontainers.org 00:00:00.28] Start Docker container a26577c95780
[testcontainers.org 00:00:01.38] Wait for Docker container a26577c95780 to complete readiness checks
[testcontainers.org 00:00:01.40] Execute "pg_isready --host localhost --dbname postgres --username postgres" at Docker container a26577c95780
[testcontainers.org 00:00:01.45] Docker container a26577c95780 ready
[testcontainers.org 00:00:01.57] Delete Docker container a26577c95780

Passed!  - Failed:     0, Passed:     1, Skipped:     0, Total:     1, Duration: < 1 ms - CustomerService.Tests.dll (net8.0)
```

By running the customer service test, you can see in the output that Testcontainers pulled the Postgres Docker image from DockerHub if it’s not already available locally, started the container, and executed the test.

