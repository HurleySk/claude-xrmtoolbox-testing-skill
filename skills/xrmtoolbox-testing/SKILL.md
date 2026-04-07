---
name: xrmtoolbox-testing
description: Test XrmToolBox plugins for Dynamics 365 / Dataverse. Use when the user wants to add tests, create mocks, write smoke tests, or validate plugin functionality without manual UI testing.
argument-hint: "[scaffold|mock|smoke|help]"
---

# XrmToolBox Plugin Testing

You are an expert at testing XrmToolBox plugins for Dynamics 365 / Dataverse. Follow the patterns and conventions below.

## Test Project Structure

XrmToolBox plugin test projects live in a `Tests/` subdirectory alongside the main plugin:

```
MyXrmToolBoxPlugin/
  MyXrmToolBoxPlugin.csproj      # Main plugin (net48, GenerateAssemblyInfo=false)
  Services/
    MyService.cs
  Tests/
    Tests.csproj                 # Test project (net48, references main plugin)
    SmokeTest.cs                 # Smoke tests for services
    Mocks/
      MockOrganizationService.cs # Mock IOrganizationService
```

**IMPORTANT**: The main plugin `.csproj` must exclude the Tests directory to prevent build conflicts:

```xml
<ItemGroup>
  <Compile Remove="Tests\**" />
  <None Remove="Tests\**" />
</ItemGroup>
```

This is required because XrmToolBox plugins use `GenerateAssemblyInfo=false` with a manual `Properties/AssemblyInfo.cs`. Without the exclusion, the SDK-style project glob picks up test files and causes duplicate assembly attribute errors.

## Commands

Based on the argument provided:

### `scaffold`

Create a test project for an existing XrmToolBox plugin:

1. **Create the Tests directory and project file**:

```xml
<!-- Tests/Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net48</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyXrmToolBoxPlugin.csproj" />
  </ItemGroup>
</Project>
```

2. **Add the exclusion to the main plugin csproj** (see above).

3. **Create a basic smoke test file** with the test runner pattern:

```csharp
using System;

namespace MyPlugin.Tests
{
    class SmokeTest
    {
        static int _passed = 0;
        static int _failed = 0;

        static void Assert(bool condition, string name)
        {
            if (condition) { Console.WriteLine($"  [PASS] {name}"); _passed++; }
            else { Console.WriteLine($"  [FAIL] {name}"); _failed++; }
        }

        static void Main(string[] args)
        {
            Console.WriteLine("=== Plugin Smoke Tests ===\n");

            // Add test methods here
            TestExample();

            Console.WriteLine($"\n=== Results: {_passed} passed, {_failed} failed ===");
            Environment.Exit(_failed > 0 ? 1 : 0);
        }

        static void TestExample()
        {
            Console.WriteLine("Test: Example");
            Assert(true, "Placeholder test");
        }
    }
}
```

4. **Build and run**:
```bash
cd Tests
dotnet build Tests.csproj --configuration Release
.\bin\Release\net48\Tests.exe
```

Note: Always build the test project by specifying `Tests.csproj` explicitly. Running `dotnet build` from the Tests directory without specifying the project file can trigger a rebuild of the parent plugin project, which may fail if the Tests source files interfere with the parent's assembly info generation.

### `mock`

Create a mock `IOrganizationService` for unit testing without a Dataverse connection.

#### Basic Mock

```csharp
using Microsoft.Xrm.Sdk;
using System;
using System.Collections.Generic;

namespace MyPlugin.Tests.Mocks
{
    /// <summary>
    /// Configurable mock that records calls and returns preset responses.
    /// </summary>
    class MockOrganizationService : IOrganizationService
    {
        public List<OrganizationRequest> ExecutedRequests { get; } = new List<OrganizationRequest>();
        public Queue<OrganizationResponse> QueuedResponses { get; } = new Queue<OrganizationResponse>();
        public Queue<Exception> QueuedExceptions { get; } = new Queue<Exception>();

        public OrganizationResponse Execute(OrganizationRequest request)
        {
            ExecutedRequests.Add(request);

            if (QueuedExceptions.Count > 0)
                throw QueuedExceptions.Dequeue();

            if (QueuedResponses.Count > 0)
                return QueuedResponses.Dequeue();

            return new OrganizationResponse();
        }

        // Stub implementations for other IOrganizationService members
        public Guid Create(Entity entity) => Guid.NewGuid();
        public void Update(Entity entity) { }
        public void Delete(string entityName, Guid id) { }
        public Entity Retrieve(string entityName, Guid id, Microsoft.Xrm.Sdk.Query.ColumnSet columnSet)
            => new Entity(entityName, id);
        public EntityCollection RetrieveMultiple(Microsoft.Xrm.Sdk.Query.QueryBase query)
            => new EntityCollection();
        public void Associate(string entityName, Guid entityId, Relationship relationship,
            EntityReferenceCollection relatedEntities) { }
        public void Disassociate(string entityName, Guid entityId, Relationship relationship,
            EntityReferenceCollection relatedEntities) { }
    }
}
```

#### Mock for AssociateRequest Testing

For plugins that create associations (N:N relationships), configure the mock to simulate duplicates and transient errors:

```csharp
// Simulate duplicate key error
var mock = new MockOrganizationService();
mock.QueuedExceptions.Enqueue(
    new System.ServiceModel.FaultException<OrganizationServiceFault>(
        new OrganizationServiceFault { Message = "Cannot insert duplicate key" }));

// Simulate transient error then success
mock.QueuedExceptions.Enqueue(new Exception("429 Too Many Requests"));
mock.QueuedResponses.Enqueue(new OrganizationResponse()); // succeeds on retry
```

#### Mock for ExecuteMultipleRequest Testing

```csharp
/// <summary>
/// Mock that handles ExecuteMultipleRequest by processing each sub-request
/// and returning an ExecuteMultipleResponse with per-item results.
/// </summary>
class ExecuteMultipleMockService : MockOrganizationService
{
    public Func<OrganizationRequest, int, OrganizationServiceFault> FaultGenerator { get; set; }

    public new OrganizationResponse Execute(OrganizationRequest request)
    {
        ExecutedRequests.Add(request);

        if (request is Microsoft.Xrm.Sdk.Messages.ExecuteMultipleRequest multiRequest)
        {
            var response = new Microsoft.Xrm.Sdk.Messages.ExecuteMultipleResponse();
            var responses = new Microsoft.Xrm.Sdk.ExecuteMultipleResponseItemCollection();

            for (int i = 0; i < multiRequest.Requests.Count; i++)
            {
                var fault = FaultGenerator?.Invoke(multiRequest.Requests[i], i);
                if (fault != null)
                {
                    responses.Add(new ExecuteMultipleResponseItem
                    {
                        RequestIndex = i,
                        Fault = fault
                    });
                }
                else
                {
                    responses.Add(new ExecuteMultipleResponseItem
                    {
                        RequestIndex = i,
                        Response = new OrganizationResponse()
                    });
                }
            }

            // Set via reflection since Responses property has no public setter
            var prop = typeof(Microsoft.Xrm.Sdk.Messages.ExecuteMultipleResponse)
                .GetProperty("Responses");
            prop.SetValue(response, responses);

            return response;
        }

        return base.Execute(request);
    }
}
```

### `smoke`

Write smoke tests for common XrmToolBox plugin components. These are lightweight, standalone tests that don't require a Dataverse connection.

#### Testing a CSV Data Loader

```csharp
static void TestCsvLoading()
{
    Console.WriteLine("Test: CSV Loading");

    var csvPath = Path.GetTempFileName();
    File.WriteAllText(csvPath,
        "guid1,guid2\n" +
        "a1b2c3d4-e5f6-7890-abcd-ef1234567890,11111111-2222-3333-4444-555555555555\n" +
        "a1b2c3d4-e5f6-7890-abcd-ef1234567891,11111111-2222-3333-4444-555555555556\n");

    var svc = new DataSourceService();
    var pairs = svc.LoadFromCsv(csvPath);

    Assert(pairs.Count == 2, "Loads 2 pairs from CSV with header");
    Assert(pairs[0].Guid1 == Guid.Parse("a1b2c3d4-e5f6-7890-abcd-ef1234567890"), "First GUID correct");

    File.Delete(csvPath);
}
```

#### Testing Deduplication

```csharp
static void TestDeduplication()
{
    Console.WriteLine("Test: Deduplication");

    var csvPath = Path.GetTempFileName();
    // Exact duplicate + reversed-order duplicate
    File.WriteAllText(csvPath,
        "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee,11111111-2222-3333-4444-555555555555\n" +
        "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee,11111111-2222-3333-4444-555555555555\n" +
        "11111111-2222-3333-4444-555555555555,aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee\n");

    var svc = new DataSourceService();
    var pairs = svc.LoadFromCsv(csvPath);

    Assert(pairs.Count == 1, "Deduplicates exact and reversed duplicates");

    File.Delete(csvPath);
}
```

#### Testing Thread-Safe Components

For components that use `Interlocked`, `ConcurrentQueue`, or locks:

```csharp
static void TestConcurrentWrites()
{
    Console.WriteLine("Test: Concurrent Writes");

    // Test whatever thread-safe component you have (e.g., ResumeTracker, counters)
    int threadCount = 10;
    int opsPerThread = 200;
    int expected = threadCount * opsPerThread;

    var threads = new Thread[threadCount];
    int counter = 0;

    for (int t = 0; t < threadCount; t++)
    {
        threads[t] = new Thread(() =>
        {
            for (int i = 0; i < opsPerThread; i++)
                Interlocked.Increment(ref counter);
        });
    }

    foreach (var t in threads) t.Start();
    foreach (var t in threads) t.Join();

    Assert(counter == expected, $"Concurrent counter: {counter} == {expected}");
}
```

#### Testing SQLite Components

If your plugin uses SQLite (e.g., for resume tracking):

```csharp
static void TestSqliteResumeTracker()
{
    Console.WriteLine("Test: SQLite Resume Tracker");

    var dbPath = Path.Combine(Path.GetTempPath(), "test_" + Guid.NewGuid().ToString("N") + ".db");

    try
    {
        var tracker = new ResumeTracker(dbPath);
        tracker.Open();

        Assert(tracker.GetCompletedCount() == 0, "Empty DB has 0 completed");

        // Track pairs and verify persistence
        for (int i = 0; i < 150; i++)
            tracker.TrackCompleted(new AssociationPair { Guid1 = Guid.NewGuid(), Guid2 = Guid.NewGuid() });
        tracker.FlushBatch();

        Assert(tracker.GetCompletedCount() == 150, "150 pairs tracked");

        // Verify persistence across reopen
        tracker.Dispose();
        var tracker2 = new ResumeTracker(dbPath);
        tracker2.Open();
        Assert(tracker2.GetCompletedCount() == 150, "Data persists after reopen");
        tracker2.Dispose();
    }
    finally
    {
        // Clean up DB + WAL/SHM files
        foreach (var ext in new[] { "", "-wal", "-shm" })
        {
            try { File.Delete(dbPath + ext); } catch { }
        }
    }
}
```

### `help`

Show this skill's available commands and XrmToolBox plugin testing guidance.

## What NOT to Test

- **WinForms UI**: Don't try to instantiate `PluginControlBase` subclasses in tests. They require the XrmToolBox hosting environment. Test the services and logic behind the UI instead.
- **MEF plugin loading**: The `[Export(typeof(IXrmToolBoxPlugin))]` attribute is validated by XrmToolBox at runtime. Testing it requires loading the full XrmToolBox process.
- **Connection management**: `ExecuteMethod()`, `WorkAsync()`, and `UpdateConnection()` are framework methods. Trust they work. Test the code that runs inside them.

## What TO Test

- **Service classes**: Any class that takes `IOrganizationService` as a parameter can be tested with the mock.
- **Data loading/parsing**: CSV parsers, FetchXML pair extraction, deduplication logic.
- **Business logic**: Validation, transformation, filtering, aggregation.
- **Resume/state tracking**: File-based or SQLite-based state that must survive restarts.
- **Concurrency**: Thread-safe counters, concurrent collections, lock correctness.
- **Error classification**: Transient vs permanent error detection, duplicate detection.
- **Retry logic**: Exponential backoff behavior, max retry limits.

## Testing Patterns

### Pattern: Test Project References Plugin Project

The test project uses `<ProjectReference>` to reference the main plugin. This gives it access to all public and internal types (if `InternalsVisibleTo` is configured).

```xml
<ItemGroup>
  <ProjectReference Include="..\MyXrmToolBoxPlugin.csproj" />
</ItemGroup>
```

### Pattern: Temp Files for Test Isolation

Always use temp files/directories for test data. Clean up in `finally` blocks:

```csharp
var tempFile = Path.GetTempFileName();
try
{
    File.WriteAllText(tempFile, testData);
    // ... run test ...
}
finally
{
    File.Delete(tempFile);
}
```

### Pattern: Simple Assert Without a Framework

XrmToolBox plugins target .NET Framework 4.8. Rather than adding NUnit/xUnit (which adds dependency complexity), use a simple console-based assertion pattern:

```csharp
static int _passed = 0, _failed = 0;

static void Assert(bool condition, string name)
{
    if (condition) { Console.WriteLine($"  [PASS] {name}"); _passed++; }
    else { Console.WriteLine($"  [FAIL] {name}"); _failed++; }
}
```

This keeps the test project zero-dependency (beyond the plugin itself) and avoids NuGet packaging complications in the XrmToolBox ecosystem.

### Pattern: Exit Code for CI Integration

Return a non-zero exit code when tests fail so CI pipelines can detect failures:

```csharp
static void Main(string[] args)
{
    RunAllTests();
    Console.WriteLine($"\n{_passed} passed, {_failed} failed");
    Environment.Exit(_failed > 0 ? 1 : 0);
}
```

### Pattern: SQLite Cleanup

SQLite with WAL mode creates `-wal` and `-shm` sidecar files. Always clean up all three:

```csharp
foreach (var ext in new[] { "", "-wal", "-shm" })
{
    try { File.Delete(dbPath + ext); } catch { }
}
```

And when deleting a SQLite database programmatically, call `SqliteConnection.ClearAllPools()` before `File.Delete()` to release pooled file handles.

## Common Mistakes to Avoid

- **Instantiating UI controls in tests**: `PluginControlBase` requires XrmToolBox's hosting. Test services, not controls.
- **Forgetting to exclude Tests/ from main csproj**: Without `<Compile Remove="Tests\**" />`, the main project picks up test .cs files and fails with duplicate assembly attribute errors.
- **Hardcoded file paths in tests**: Use `Path.GetTempFileName()` or `Path.GetTempPath()` for test data.
- **Not cleaning up temp files**: Use `try/finally` to ensure cleanup even when tests fail.
- **Testing mock behavior instead of real behavior**: The mock should simulate Dataverse responses, not become the thing you test. Test your code's reaction to mock responses.
- **Sharing state between tests**: Each test method should create its own data and clean it up. Don't rely on execution order.
- **Adding test framework NuGet packages to the plugin**: Keep test dependencies in the test project only. Never add NUnit/xUnit to the main plugin `.csproj`.
