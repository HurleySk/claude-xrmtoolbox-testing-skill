---
name: xrmtoolbox-testing
description: Test XrmToolBox plugins for Dynamics 365 / Dataverse. Use when the user wants to add tests, create mocks, write smoke tests, run UI tests, or validate plugin functionality.
argument-hint: "[scaffold|mock|smoke|ui-test|help]"
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

### `ui-test`

Run automated UI tests against a plugin using the **XrmToolBox Test Harness** and **FlaUI-MCP**. This is a multi-step agent workflow — execute each step in order, checking the result before proceeding.

The test harness ([xrmtoolbox-testing-toolkit](https://github.com/HurleySk/xrmtoolbox-testing-toolkit)) is a standalone WinForms app that hosts any XrmToolBox plugin DLL outside of XrmToolBox, injecting a configurable mock `IOrganizationService`. The harness repo also includes `generate-mockdata.ps1` which auto-analyzes plugin source code and generates starter mock data + a control inventory.

**Terminology**: Throughout this workflow, `$HARNESS_EXE`, `$HARNESS_REPO`, `$PLUGIN_DIR`, and `$PLUGIN_DLL` are conceptual variables — remember these paths as you discover them in Step 1.

#### Step 1: Verify Prerequisites

Execute these checks in order. Fix any that fail before proceeding.

**1a. Locate the test harness repo and exe.**

Glob for `XrmToolBox.TestHarness.exe` on disk:
```bash
find "$HOME/source/repos" -path "*/xrmtoolbox-testing-toolkit/*/bin/Release/*/XrmToolBox.TestHarness.exe" 2>/dev/null | head -1
```

- **Check**: The find returns a path. Remember it as `$HARNESS_EXE`. Derive `$HARNESS_REPO` as the repo root (ancestor containing `XrmToolBox.TestHarness.slnx`).
- **Fix if not found**: Clone and build:
```bash
git clone https://github.com/HurleySk/xrmtoolbox-testing-toolkit.git "$HOME/source/repos/xrmtoolbox-testing-toolkit"
cd "$HOME/source/repos/xrmtoolbox-testing-toolkit"
dotnet build --configuration Release
```
Then re-run the find to get `$HARNESS_EXE` and `$HARNESS_REPO`.

**1b. Check that FlaUI-MCP is installed and registered.**

```bash
test -f "C:/tools/FlaUI-MCP/FlaUI.Mcp.exe" && echo "INSTALLED" || echo "NOT INSTALLED"
```

- **Check**: Output says `INSTALLED`.
- **Fix if NOT INSTALLED**: Run the setup script from the harness repo:
```powershell
powershell -ExecutionPolicy Bypass -File "$HARNESS_REPO\setup-flaui-mcp.ps1"
```
This clones FlaUI-MCP, builds it, publishes to `C:\tools\FlaUI-MCP`, and registers it as the `flaui-mcp` MCP server in Claude Code. Requires .NET 8+ SDK. **Note**: If this is the first time setting up FlaUI-MCP, you may need to restart Claude Code for the MCP server to become available.

**1c. Locate the plugin DLL.**

First, find the `.csproj` file name to know the DLL name:
```bash
ls "$PLUGIN_DIR"/*.csproj
```

The plugin DLL will match the `.csproj` stem. Look for it in `bin/Release`:
```bash
find "$PLUGIN_DIR" -path "*/bin/Release/*" -name "*.dll" | grep -v "Microsoft\." | grep -v "System\." | grep -v "Newtonsoft" | grep -v "McTools" | grep -v "XrmToolBox\." | head -5
```

- **Check**: Identify the plugin DLL (matches the `.csproj` file name, e.g., `MyPlugin.csproj` → `MyPlugin.dll`).
- **Fix if not found**: Build the plugin:
```bash
dotnet build "$PLUGIN_DIR"/*.csproj --configuration Release
```
- Remember the full path as `$PLUGIN_DLL`.

#### Step 2: Analyze the Plugin

Use the `generate-mockdata.ps1` script to automatically discover controls, SDK patterns, and entity names.

**2a. Run the generator script.**

```powershell
powershell -ExecutionPolicy Bypass -File "$HARNESS_REPO\generate-mockdata.ps1" -PluginSourceDir "$PLUGIN_DIR"
```

This produces two files in the plugin directory:
- `test-mockdata.json` — starter mock data with responses for all discovered SDK patterns
- `test-control-inventory.json` — all UI controls with names, types, and interaction categories

**2b. Read the generated files.**

Read `$PLUGIN_DIR/test-control-inventory.json`. Key fields:
- `controls[]` — array of `{name, type, category, prefix}` for every UI control
- `summary` — counts by category (buttons, grids, dropdowns, textBoxes, etc.)
- `actionButtons[]` — list of all button names
- `primaryAction` — the recommended first button to click (heuristic: first btn* that isn't stop/cancel/close/browse)

Read `$PLUGIN_DIR/test-mockdata.json`. It contains:
- WhoAmI response (always present)
- Execute responses for each discovered SDK request type
- RetrieveMultiple responses for each discovered entity name
- Wildcard fallbacks for all operations

**2c. Review and augment if needed.**

The generated mock data covers discovered patterns. You may need to add entries if:
- The plugin needs specific attribute values (e.g., a grid expects `friendlyname` on solutions)
- The plugin uses metadata requests that need non-empty results
- The plugin uses FetchXML with specific expected columns
- Error appears when running: check `calls.json` for unmatched calls and add matching responses

#### Step 3: Launch the Test Harness

Run the harness in the background. The `--record` path is relative to where you launch the command:
```bash
"$HARNESS_EXE" --plugin "$PLUGIN_DLL" --mockdata "$PLUGIN_DIR/test-mockdata.json" --screenshots "$PLUGIN_DIR/screenshots" --record "$PLUGIN_DIR/calls.json" &
```

Wait 3-5 seconds for initialization, then verify via FlaUI-MCP: call `windows_list_windows` and look for a window with title containing `"Test Harness"`. Remember its handle (e.g., `w1`).

If found, the plugin loaded successfully.

**Common failures:**
- **Plugin DLL dependencies missing**: Copy dependency DLLs from harness output dir alongside plugin DLL, or vice versa
- **JSON syntax error**: Validate `test-mockdata.json` is valid JSON
- **Plugin throws on load**: Check console output; the plugin may require specific mock data to be present during initialization (e.g., WhoAmI)

#### Step 4: Drive the UI via FlaUI-MCP

FlaUI-MCP uses a snapshot-and-ref model: call `windows_snapshot` to get the accessibility tree with element refs (e.g., `w1e5`), then use those refs with `windows_click`, `windows_fill`, `windows_get_text`, etc. **Element refs change after each snapshot**, so always re-snapshot after UI changes.

**4a. Take initial screenshot and snapshot.** Use `windows_screenshot` then `windows_snapshot` on the harness window handle. The snapshot returns a text tree of all controls with their names and refs. Cross-reference against your control inventory from Step 2b — verify key controls are present.

**4b. Set up input controls** (from the inventory, category = input-*):
- **Dropdowns (cbo/cmb)**: `windows_click` to expand, re-snapshot to see items, `windows_click` on first item
- **TextBoxes (txt)**: `windows_fill` with a test value appropriate to the control name (e.g., `txtFilter` → `"test"`)
- **CheckBoxes (chk)**: `windows_click` to toggle if needed
- **Numerics (nud)**: Leave at defaults unless testing specific values

**4c. Click the primary action button.** Use the `primaryAction` field from the control inventory. Find it in the snapshot and `windows_click` it. Common names: `btnLoad`, `btnLoadEntities`, `btnStart`, `btnExport`, `btnExecute`, `btnRetrieve`.

**4d. Wait for completion.** After clicking, wait 2-3 seconds then re-snapshot and check:
- Has a DataGridView been populated? (Look for row elements under the datagrid in the snapshot)
- Has a status label text changed? Use `windows_get_text` to read it
- Is the action button still present and not marked `[disabled]` in the snapshot?

**4e. Take post-action screenshot.** Use `windows_screenshot` on the harness window.

#### Step 5: Verify Results

**5a. Check UI state via FlaUI-MCP:**
- Read text from status labels with `windows_get_text`
- Check for modal error dialogs with `windows_list_windows` — look for new windows with "Error" in the title
- Re-snapshot with `windows_snapshot` and look at the datagrid tree for row data

**5b. Close the harness and check SDK call recording.**

Close the harness with `windows_close` so it writes `calls.json`.

Then read `$PLUGIN_DIR/calls.json`. Each entry has:
- `Operation` — the SDK method called (Create, Retrieve, RetrieveMultiple, Update, Delete, Execute, Associate, Disassociate)
- `EntityName` — the entity targeted (if applicable)
- `RequestTypeName` — the full type name for Execute calls
- `WasMatched` — whether a mock response was found
- `MatchedDescription` — which mock entry matched

Verify:
- Expected operations were called (e.g., plugin that loads entities should have RetrieveMultiple or RetrieveAllEntitiesRequest)
- No unexpected unmatched calls (check `WasMatched: false` entries)

**Debugging loop**: If there are unmatched calls, add matching responses to `test-mockdata.json` and re-run from Step 3.

#### Step 6: Report

Summarize the test run:
- **Plugin loaded**: yes/no
- **Primary action completed**: yes/no (based on UI state changes)
- **SDK calls**: total count, breakdown by operation, any unmatched
- **Errors**: any error dialogs or exceptions
- **Screenshots**: paths to initial and post-action screenshots
- **Assessment**: pass (plugin loaded, action completed, no errors) or fail (with reason)

#### Appendix: Mock Data Match Criteria

| Key | Description |
|-----|-------------|
| `entityName` | Match by entity logical name |
| `requestType` | Match Execute requests by full type name (e.g., `Microsoft.Crm.Sdk.Messages.WhoAmIRequest`) |
| `queryExpressionEntity` | Match QueryExpression by entity name |
| `fetchXmlContains` | Match FetchExpression containing a substring |
| `*` | Wildcard — matches anything |

Responses are matched in order; first match wins. Use `resultsFile` for large payloads (metadata responses) in separate JSON files.

#### Appendix: CLI Options

| Flag | Description | Default |
|------|-------------|---------|
| `--plugin, -p <path>` | Plugin DLL path (required) | |
| `--mockdata, -m <path>` | Mock data JSON config | (empty responses) |
| `--width <px>` | Window width | 1024 |
| `--height <px>` | Window height | 768 |
| `--screenshots, -s <dir>` | Screenshot output directory | |
| `--org <name>` | Organization display name | Mock Organization |
| `--record, -r <path>` | Record SDK calls to JSON on exit | |
| `--no-autoconnect` | Don't inject mock service on load | |

#### Appendix: Common Request Type Full Names

| Short Name | Full Type Name |
|------------|---------------|
| WhoAmIRequest | Microsoft.Crm.Sdk.Messages.WhoAmIRequest |
| RetrieveAllEntitiesRequest | Microsoft.Xrm.Sdk.Messages.RetrieveAllEntitiesRequest |
| RetrieveEntityRequest | Microsoft.Xrm.Sdk.Messages.RetrieveEntityRequest |
| RetrieveAttributeRequest | Microsoft.Xrm.Sdk.Messages.RetrieveAttributeRequest |
| ExecuteMultipleRequest | Microsoft.Xrm.Sdk.Messages.ExecuteMultipleRequest |
| AssociateRequest | Microsoft.Xrm.Sdk.Messages.AssociateRequest |
| RetrieveRelationshipRequest | Microsoft.Xrm.Sdk.Messages.RetrieveRelationshipRequest |

#### Appendix: Control Name Prefixes (Hungarian Notation)

| Prefix | Control Type | FlaUI-MCP Tool |
|--------|-------------|---------------|
| btn | Button | `windows_click` |
| dgv | DataGridView | `windows_snapshot` (read tree), `windows_get_text` (cells) |
| cbo, cmb | ComboBox | `windows_click` (expand + select item) |
| txt | TextBox | `windows_fill` |
| chk | CheckBox | `windows_click` (toggle) |
| nud | NumericUpDown | `windows_fill` |
| rtb | RichTextBox | `windows_fill` |
| lbl | Label | `windows_get_text` |
| tab | TabControl | `windows_click` (tab header) |
| progress | ProgressBar | `windows_get_text` |

#### Appendix: Tips

- Press **F12** in the harness window to take a manual screenshot
- The mock service records every SDK call — `calls.json` is written when the harness window closes
- Use `"fault"` entries in mock data to test error handling paths
- Use `"delay"` to simulate slow responses and test loading indicators

### `help`

Show this skill's available commands and XrmToolBox plugin testing guidance.

## What NOT to Test

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
- **UI behavior** (via Test Harness): Button enable/disable states, grid population, control visibility, error dialogs, loading indicators. Use the `ui-test` command for guidance.

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
