# Memory Leak Analysis - Complete Instructions

## Part 1: Running the Sample Code

### Build and Run
```bash
cd src/MemoryLeakSample
dotnet build -c Release
dotnet run
```

The application will start and display memory leak simulation progress. Keep it running while you proceed to generate a dump.

### What's Happening
The application intentionally creates 3 types of memory leaks:
1. **Static Collection Leak**: 1000 × 1MB objects stored in static list (never cleared)
2. **Event Handler Leak**: 500 event subscribers never unsubscribed
3. **Unmanaged Resources Leak**: 100 FileStreams never disposed

**Memory Usage**: Expect to see memory grow to 1-2 GB+ as the application runs.

---

## Part 2: Generating Memory Dump Files

### Windows - Using Task Manager
1. Run the MemoryLeakSample application
2. Open **Task Manager** (Ctrl+Shift+Esc)
3. Find **MemoryLeakSample.exe** in the Processes tab
4. Right-click → **Create dump file**
5. Save location: `C:\Users\[YourUser]\AppData\Local\Temp\MemoryLeakSample.exe_[timestamp].dmp`

### Windows - Using WinDbg
```bash
# Install WinDbg (Windows Debugging Tools)
# Get the process ID
tasklist | findstr MemoryLeakSample

# Create dump (replace PID)
procdump -ma 12345 dump.dmp
```

### Windows - Using .NET Diagnostic Tools
```bash
# Install dotnet-dump tool
dotnet tool install -g dotnet-dump

# Get process ID
dotnet-dump ps | findstr MemoryLeakSample

# Generate dump (replace PID)
dotnet-dump collect -p 12345 -o ./dump.dmp
```

### Linux/macOS - Using dotnet-dump
```bash
# Install tool
dotnet tool install -g dotnet-dump

# Get process ID
ps aux | grep MemoryLeakSample

# Generate dump (replace PID)
dotnet-dump collect -p 12345 -o ./dump.dmp
```

### Docker - Generate dump inside container
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0
WORKDIR /app
COPY . .
RUN dotnet build -c Release
RUN dotnet tool install -g dotnet-dump

CMD dotnet run & sleep 10 && dotnet-dump collect -p $! -o /dumps/memleak.dmp
```

---

## Part 3: Analyzing Dumps with AI

### Understanding the Dump File
A memory dump contains:
- **Heap objects** - All objects currently in memory
- **Object references** - What's holding references to objects
- **Type information** - Class names, field names, methods
- **String data** - Text content of strings in memory

### AI Analysis Approach

#### 1. **Pattern Recognition**
Ask AI to identify:
```
"Analyze this memory dump and identify:
- Large allocations (>10MB)
- Collections with many items
- Objects with high reference counts
- Potential circular references"
```

#### 2. **Static Collection Detection**
```
"Look for private static List or Dictionary fields with thousands of items.
These are likely memory leaks because they never release objects."
```

#### 3. **Event Handler Leaks**
```
"Find event subscriptions that are never unsubscribed.
Look for delegates held by event fields with increasing counts."
```

#### 4. **Unmanaged Resource Leaks**
```
"Identify FileStream, DbConnection, or other IDisposable objects
that are not disposed and not in using statements."
```

#### 5. **Generational Analysis**
```
"Objects in Gen 2 (old generation) suggest long-lived objects
or objects that should have been collected."
```

---

## Using GitHub Copilot to Analyze

### Copilot Chat Prompts

**Prompt 1: Load and parse dump**
```
I have a .NET memory dump file at ./dump.dmp
Can you write C# code using ClrMD to:
1. Load the dump
2. Print all objects larger than 1MB
3. Show what's referencing them
```

**Prompt 2: Find specific leak patterns**
```
Write a ClrMD analysis that finds:
1. Static fields containing collections with >100 items
2. Event handlers subscribed but never unsubscribed
3. Disposed objects still in memory
```

**Prompt 3: Generate leak report**
```
Given a memory dump, generate a report showing:
- Total memory used by type
- Top 10 largest objects
- Suggested root causes of leaks
- Recommended fixes
```

---

## Using Free AI Tools for Analysis

### Option 1: ChatGPT / Claude
1. Export heap dump as text (if tools allow)
2. Paste relevant sections to ChatGPT/Claude
3. Ask:
```
"I have a memory dump with these large objects:
- System.Collections.Generic.List<byte[]> with 1000 items, 1GB total
- 500 event handlers in EventHolder.DataReceived
- 100 FileStream objects never disposed

What are the likely causes and fixes?"
```

### Option 2: Use ClrMD with AI Integration
```csharp
// Use ClrMD to analyze dump
var runtime = DataTarget.LoadCrashDump(dumpPath).ClrVersions[0].CreateRuntime();
var heap = runtime.Heap;

// Collect suspicious patterns
var largeObjects = heap.GetObjectsOfType("System.Byte[]")
    .Where(obj => obj.Size > 1_000_000)
    .ToList();

// Send to AI for analysis
string analysisPrompt = $@"
Analyze this memory leak:
- Found {largeObjects.Count} byte arrays larger than 1MB
- Total size: {largeObjects.Sum(o => o.Size) / 1_000_000}MB
- Stored in: System.Collections.Generic.List<byte[]> 

What's the likely cause?";

// Call your AI API with prompt...
```

### Option 3: Hugging Face Inference
```csharp
using var client = new HttpClient();
var prompt = "Analyze memory leak: large static collections not cleared";

var request = new HttpRequestMessage(HttpMethod.Post, 
    "https://api-inference.huggingface.co/models/meta-llama/Llama-2-7b-chat");

request.Headers.Add("Authorization", $"Bearer {apiKey}");
request.Content = new StringContent(
    $