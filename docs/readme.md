## Better Code Box

A local, extensible, and powerful tool for finding issues and code smells in your C# codebase.

**No Cloud Tracking. No Data Mining. No Ads. No Cost.**

Better Code Box helps you keep your code maintainable, readable, and high-quality by running a suite of static analyzers against your .NET solution and producing a structured JSON report.

## [Latest Release](https://github.com/Box-In-A-Box-Studios/BetterCodeBox/releases/latest)

---

## Getting Started

### Requirements

- [.NET 6 SDK](https://dotnet.microsoft.com/download) or later
- A .NET solution directory to analyze

### Run via CLI

```bash
dotnet BetterCodeBox.dll <path-to-your-solution-directory>
```

Replace `<path-to-your-solution-directory>` with the root folder of your solution (the folder containing your `.sln` file and project directories).

### Run via Docker

```bash
docker run --rm \
  -v /path/to/your/solution:/solution \
  -v /path/to/output:/app/results \
  bettercodeboximage /solution
```

Results are written to a `results/` directory (or the directory specified by `ResultDirectory` in your config).

---

## Configuration

Create an `appsettings.json` file alongside the executable to customize analyzer thresholds:

```json
{
  "ResultDirectory": "results",
  "MaxLines":        250,
  "MaxDepth":        5,
  "MaxBlocks":       10,
  "MaxParameters":   4
}
```

| Key               | Default | Description                                                                                   |
|-------------------|---------|-----------------------------------------------------------------------------------------------|
| `ResultDirectory` | `results` | Directory where `results.json` is written                                                   |
| `MaxLines`        | `250` (file), `100` (method) | Maximum allowed lines. Defaults to **250** for the file length check and **100** for the method length check when the key is absent. |
| `MaxDepth`        | `5`     | Maximum allowed nesting depth inside a method                                                 |
| `MaxBlocks`       | `10`    | Maximum allowed number of block nodes (branches) in a method                                  |
| `MaxParameters`   | `4`     | Maximum allowed parameters per method                                                         |

All keys are optional. If a key is absent, the default value shown above is used.

---

## Analyzers

Better Code Box runs the following analyzers against all non-test C# files in your solution:

### 1. File Length (`ClassLengthAnalyzer`)

Reports every source file and its total line count. Files that exceed **`MaxLines`** (default: 250) are flagged as errors.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Full path to the source file         |
| `Value`            | Total line count                     |
| `Type`             | `Success` if under threshold, `Error` if over |

---

### 2. Method Length (`MethodLengthAnalyzer`)

Parses every method in non-test files using Roslyn and reports its line count. Methods exceeding **`MaxLines`** (default: 100) are flagged as errors.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Fully-qualified method name          |
| `Value`            | Line count of the method             |
| `Type`             | `Success` if under threshold, `Error` if over |

> **Note:** If a method is overridden in the same file, the override replaces the base entry.

---

### 3. Parameters Per Method (`ParametersPerMethodAnalyzer`)

Counts the number of parameters declared on each method. Methods with more than **`MaxParameters`** (default: 4) parameters are flagged as errors.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Fully-qualified method name          |
| `Value`            | Number of parameters                 |
| `Type`             | `Success` if at or under threshold, `Error` if over |

---

### 4. Method Block Depth (`MethodMaxDepthAnalyzer`)

Calculates the maximum nesting depth of block statements (e.g. `if`, `for`, `while`, `try`) inside each method. Methods with a depth greater than **`MaxDepth`** (default: 5) are flagged as errors.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Fully-qualified method name          |
| `Value`            | Maximum nesting depth                |
| `Type`             | `Success` if at or under threshold, `Error` if over |

---

### 5. Method Block Complexity (`MethodMaxBlockAnalyzer`)

Counts the total number of block nodes inside each method as a proxy for cyclomatic / branch complexity. Methods with more than **`MaxBlocks`** (default: 10) blocks are flagged as errors.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Fully-qualified method name          |
| `Value`            | Total block count                    |
| `Type`             | `Success` if at or under threshold, `Error` if over |

---

### 6. Duplicate Code (`MethodDuplicationAnalyzer`)

Compares every pair of methods (with more than 3 lines) across all non-test files. Comments, whitespace, and string literals are stripped before comparison. Method pairs with a similarity score above **0.8** (80 %) are reported as errors.

| Output field         | Description                                        |
|----------------------|----------------------------------------------------|
| `Identifier`         | Fully-qualified name of the first method           |
| `SecondaryIdentifier`| Fully-qualified name of the second (duplicate) method |
| `Value`              | Similarity score (0.0 – 1.0)                       |
| `Type`               | Always `Error`                                     |

---

### 7. Test Coverage (`TestCoverageAnalyzer`) ⚠️ *Experimental*

Attempts to detect which production methods are directly invoked from test files by performing a name-based static match. Methods with no matching test invocation are flagged as uncovered.

> **Not ready for production use.** This analyzer uses simple identifier matching and does not perform semantic analysis. It may produce false positives and false negatives.

| Output field       | Description                          |
|--------------------|--------------------------------------|
| `Identifier`       | Fully-qualified method name          |
| `Value`            | `Covered` or `Not Covered`           |
| `Type`             | `Success` if covered, `Error` if not |

---

## Output Format

All results are written to `results/results.json` (or the path set in `ResultDirectory`). The file has the following structure:

```json
{
  "Date": "2024-01-15 10:30:00",
  "Version": "1.0",
  "Analyzers": [
    {
      "Title": "Files over 250 lines",
      "FileTitle": "files-over-250-lines",
      "Results": [
        {
          "Identifier": "src/MyService.cs",
          "Type": "Error",
          "Value": "312",
          "SecondaryIdentifier": null
        }
      ]
    }
  ]
}
```

- **`Title`** – Human-readable name of the analyzer.
- **`FileTitle`** – URL/filename-safe version of the title.
- **`Results`** – Array of findings, sorted by severity (worst first).
  - **`Identifier`** – File path or fully-qualified method name.
  - **`Type`** – `Success`, `Warning`, or `Error`.
  - **`Value`** – The measured value (line count, depth, similarity score, etc.).
  - **`SecondaryIdentifier`** – Second method name (only used by the duplicate code analyzer).

---

## Project Detection

Better Code Box automatically separates test projects from production projects by scanning `.csproj` files for the following test framework references:

- `Microsoft.NET.Test.Sdk`
- `NUnit`
- `xunit`

Analyzers run only against **production** (non-test) source files. The test coverage analyzer additionally reads test files to find method invocations.

The following paths and files are always excluded from analysis:

- `bin\`, `obj\`, `Properties\`
- `AssemblyInfo.cs`, `Usings.cs`

---

## Planned Future Analyzers

The following analyzers are planned for future releases:

- Naming conventions (variables, methods, classes)
- Magic numbers and hardcoded strings / paths / URLs / secrets
- Unused variables, methods, parameters, and imports
- Empty classes and empty non-virtual methods
- Missing default case in switch statements
- Folder/namespace alignment
- Excessive use of `var`, `dynamic`, or `out`/`ref` parameters
- Nested loops and nested try/catch
- Async/await best practices (`async void`, missing `await`, `Thread.Sleep`)
- Null-check and nullability suggestions
- Public static mutable variables
- Git repository health checks (README, license, `.gitignore`)
- Dockerfile, Docker Compose, and `.gitignore` generators