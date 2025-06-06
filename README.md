# Stelp - Starlark Event and Line Processor

A command-line tool that processes text streams using [Starlark](https://github.com/bazelbuild/starlark) scripts (a Python-like configuration language). Transform, filter, and analyze data with familiar Python syntax in streaming pipelines.

## Quick Start

```bash
# Basic text transformation
echo "hello world" | stelp -e 'line.upper()'
# → HELLO WORLD

# Multi-step pipeline: filter then transform
seq 1 10 | stelp --filter 'int(line) % 2 == 0' -e '
count = inc("even_numbers")
f"Even #{count}: {line}"
'
# → Even #1: 2
# → Even #2: 4
# → Even #3: 6
# → Even #4: 8
# → Even #5: 10
```

## Features

- **Pipeline Processing**: Chain multiple transformation and filter steps
- **Python-like Syntax**: Familiar Starlark (Python subset) scripting
- **Data Format Support**: JSON, CSV, key-value, field splitting
- **Global State**: Accumulate counters, track state across lines
- **Side Effects**: Emit additional output, skip lines, early exit
- **Meta Variables**: Access line numbers, filenames, record counts

## Usage

```bash
stelp [OPTIONS] [FILES...]

Options:
  -e, --eval <EXPRESSION>     Pipeline evaluation expressions (executed in order)
      --filter <EXPRESSION>   Filter expressions (only keep lines where true)
  -s, --script <FILE>         Script file containing pipeline definition
  -I, --include <FILE>        Include Starlark files (processed in order)
  -o, --output <FILE>         Output file (default: stdout)
      --debug                 Debug mode - show processing details
      --fail-fast             Fail on first error instead of skipping lines
```

## Core Concepts

### Line-by-Line Processing
Each line becomes the `line` variable in your script:
```python
line.upper()                    # Transform line
len(line) > 10                  # Filter condition
f"Processed: {line}"            # Format output
```

### Pipeline Stages
Commands execute in the order specified:
```bash
cat README.md | stelp --filter 'len(line) > 3' -e 'line.upper()' -e 'f"Result: {line}"'
# 1. Filter: keep lines longer than 3 chars
# 2. Transform: convert to uppercase  
# 3. Format: add prefix
```

### F-String Limitations
**Important**: F-strings only work with atomic values (plain variable names). Complex expressions won't work.

✅ **Correct f-string usage:**
```python
# Extract to variables first
user = data["user"]
count = glob.get("counter", 0)
f"User: {user}, Count: {count}"

# Use with atomic variables  
f"Line {LINENUM}: {line}"
f"File: {FILENAME}"
```

❌ **Incorrect f-string usage:**
```python
# Don't use complex expressions in f-strings
f"User: {data['user']}"           # Won't work - dict access
f"Count: {glob.get('counter')}"   # Won't work - function call
f"Length: {len(line)}"            # Won't work - function call
```

**Workaround**: Always extract complex expressions to simple variables first:
```python
# Instead of: f"User: {data['user']} has {len(items)} items"
user = data["user"]
item_count = len(items)
f"User: {user} has {item_count} items"
```

### Control Flow Functions

```python
emit("message")           # Output additional line (continues processing)
skip()                    # Skip current line (no output)
exit("reason")           # Stop processing with message
inc("counter")           # Increment counter, returns new value
```

## Examples

### Basic Text Processing
```bash
# Transform case
stelp -e 'line.upper()' input.txt

# Filter and count
stelp --filter 'len(line) > 50' -e '
count = inc("long_lines")
line_len = len(line)
f"Long line #{count} ({line_len} chars): {line}"
' input.txt

# Regex processing
stelp -e 'regex_replace(r"[ERROR]", "[🔴]", line)' \
      -e 'f"[{LINENUM}] {line}"' \
      error.log
```

### Global State & Counting
```bash
stelp -e '
count = inc("total")
if "ERROR" in line:
    error_count = inc("errors")
    emit(f"🚨 Error #{error_count}: {line}")
f"[{count}] {line}"
' server.log
```

### JSON Processing
```bash
# Extract and transform JSON fields
echo '{"user": "alice", "action": "login"}' | \
stelp -e '
data = parse_json(line)
user = data["user"]
action = data["action"]
f"{user} performed {action}"
'
```

### CSV Processing  
```bash
# Process CSV with headers
stelp -e '
if LINENUM == 1:
    line  # Keep header
else:
    fields = parse_csv(line)
    age = int(fields[1])  # Age column
    if age >= 18:
        name = fields[0]
        dump_csv([name, "adult"])
    else:
        skip()
' users.csv
```

### Log Analysis with Early Exit
```bash
stelp -e '
if "FATAL" in line:
    emit(f"💀 Fatal error at line {LINENUM}: {line}")
    exit("Processing stopped due to fatal error")
elif "ERROR" in line:
    error_count = inc("errors")
    f"Error #{error_count}: {line}"
else:
    line
' application.log
```

### Multi-file Processing
```bash
# Process multiple files with accumulated state
stelp -e '
file_lines = inc(f"lines_{FILENAME}")
total_lines = inc("total_lines")

f"[{FILENAME}:{LINENUM}] (file: {file_lines}, total: {total_lines}) {line}"
' file1.txt file2.txt file3.txt
```

## Script Files

For complex processing, use script files:

```python
# process_logs.star
def categorize_level(line):
    if "ERROR" in line:
        return "error"
    elif "WARN" in line:
        return "warning"  
    else:
        return "info"

# Main processing
category = categorize_level(line)
category_count = inc(f"{category}_count")

if category == "error":
    emit(f"🔴 Error #{category_count}: {line}")

# Extract to variables for f-string
category_upper = category.upper()
f"[{category_upper}:{category_count}] {line}"
```

```bash
stelp -s process_logs.star server.log
```

## Context Variables

Context variables provide information about the current processing state:

```python
LINENUM           # Current line number (1-based)
FILENAME          # Current filename or None for stdin
RECNUM            # Record number within current file (1-based)
```

Use directly in f-strings or expressions:
```python
# These work because they're atomic variables
f"Line {LINENUM} in {FILENAME}: {line}"
f"Processing record {RECNUM}"
```

## Variable Scopes

- **Local variables**: Reset for each line (`parts = line.split()`)
- **Global variables**: Persist across lines using `glob` dictionary (`glob["key"] = value`)  
- **Counters**: Increment with `inc("counter")` (returns new value)
- **Meta variables**: Context information (`LINENUM`, `FILENAME`, `RECNUM`)
- **Shared functions**: Defined in include files (`-I utils.star`)

## Exit Codes

- `0`: Success (some output produced)
- `1`: Processing errors occurred  
- `2`: No output produced

## Installation

```bash
git clone <repository>
cd stelp  
cargo build --release
# Binary: target/release/stelp
```

## Examples

```bash
# Generate test data
seq 1 100 | stelp -e 'f"Item {line}: {LINENUM}"'

# Parse Apache logs
stelp -e 'regex_replace(r"(\d+\.\d+\.\d+\.\d+).*", r"IP: \1", line)' access.log

# Count patterns across files  
stelp -e '
if regex_match(r"ERROR", line):
    error_count = inc("errors")
else:
    error_count = glob.get("errors", 0)
    
f"Total errors so far: {error_count}"
' *.log

# CSV transformation
echo -e "name,age\nAlice,25\nBob,30" | stelp -e '
if LINENUM == 1:
    line + ",category"
else:
    fields = parse_csv(line)
    age = int(fields[1])
    category = "senior" if age >= 30 else "junior"
    name = fields[0]
    age_str = fields[1]
    dump_csv([name, age_str, category])
'
```

## License

MIT