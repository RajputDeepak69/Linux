# AWK — Learning Notes

`awk` is a powerful text-processing and pattern-scanning tool commonly used in Linux/Unix environments for searching, filtering, extracting, and manipulating structured text.

It is especially useful when working with log files, command outputs, delimited text (e.g., CSV, TSV), and column-oriented tabular data.

---

## 1. Basic Syntax

```bash
awk 'pattern { action }' file
```

### Example: Print the First Column
```bash
awk '{print $1}' file.txt
```
> Extracts and displays the first column from each line of `file.txt`.

---

## 2. How AWK Thinks

AWK reads input **line by line** (record by record) and automatically divides each line into **fields** (columns) separated by whitespace by default.

### Mental Model

```text
Line:   apple       100         red
          │          │           │
Field:   $1         $2          $3
```

### Useful Built-in Variables

| Variable | Description |
| :--- | :--- |
| `$0` | The entire current line |
| `$1` | First field |
| `$2` | Second field |
| `$NF` | Last field in the current line |
| `NF` | Total number of fields in the current line |
| `NR` | Current line number (record number) |
| `FS` | Input field separator (default: whitespace) |
| `OFS` | Output field separator (default: space) |

---

## 3. Basic Examples

### Print Specific Columns
```bash
# Print first column
awk '{print $1}' file.txt

# Print second column
awk '{print $2}' file.txt

# Print multiple columns (1st and 3rd)
awk '{print $1, $3}' file.txt

# Print entire line
awk '{print $0}' file.txt
```

### Print Line Numbers
```bash
awk '{print NR, $0}' file.txt
```

**Output:**
```text
1 first line
2 second line
3 third line
```

---

## 4. Filtering Data

AWK can evaluate conditions before deciding to execute an action:

```bash
# Numerical condition: print lines where 2nd field is greater than 100
awk '$2 > 100 {print $1, $2}' file.txt

# String equality match:
awk '$1 == "John" {print $0}' file.txt

# Regular expression pattern match:
awk '/error/ {print $0}' logfile.txt
```

---

## 5. Using a Custom Delimiter

By default, AWK splits fields on spaces and tabs. Use the `-F` flag to define custom delimiters.

### CSV Files (Comma-separated)
```bash
awk -F',' '{print $1, $3}' file.csv
```

### System Files (e.g., `/etc/passwd` colon-separated)
```bash
awk -F':' '{print $1, $7}' /etc/passwd
```

---

## 6. Variables and Calculations

AWK natively supports arithmetic operations and accumulator variables.

### Sum a Column
```bash
awk '{sum += $2} END {print sum}' file.txt
```

### Calculate an Average
```bash
awk '{sum += $2} END {print sum / NR}' file.txt
```

---

## 7. `BEGIN` and `END` Blocks

AWK provides two special blocks that run outside the normal line-processing loop:

* **`BEGIN`**: Runs **before** any input lines are processed (ideal for setting headers or variables).
* **`END`**: Runs **after** all lines have been processed (ideal for printing summaries and totals).

```bash
# Example with both blocks
awk 'BEGIN {print "Starting"} {print $1} END {print "Finished"}' file.txt

# Printing total sum after calculation
awk '{sum += $2} END {print "Total:", sum}' file.txt
```

### Execution Flow

```text
       BEGIN block
            ↓
  [For each input line]
            ↓
      Match pattern?
            ↓
      Perform action
            ↓
        END block
```

---

## 8. AWK in Linux Pipelines

`awk` is commonly chained with other Linux CLI tools:

```bash
# Extract user, PID, and command name from process list
ps aux | awk '{print $1, $2, $11}'

# Extract filesystem name and usage percentage
df -h | awk '{print $1, $5}'
```

---

## 9. Tool Comparison: `grep` vs `sed` vs `awk`

| Tool | Primary Purpose | Mental Model |
| :--- | :--- | :--- |
| **`grep`** | Search & filter lines | **Find** |
| **`sed`** | Stream edit & substitute text | **Replace / Edit** |
| **`awk`** | Parse structured columns & compute | **Extract / Calculate** |

### Combining Them Together
```bash
grep "ERROR" logfile.txt | awk '{print $1, $5}'
```

---

## 10. Summary Cheat Sheet

### Core Execution Syntax
```bash
awk 'condition { action }' file
```

### Essential Field Shortcuts
```text
$0   → Entire line
$1   → First field
$2   → Second field
$NF  → Last field
NF   → Field count
NR   → Line number
```

> **Formula to Remember:**  
> **`AWK`** = Pattern Matching + Column Extraction + Text Processing + Calculations