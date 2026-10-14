# Linux `sed` (Stream Editor) — Cheat Sheet & Reference Guide

`sed` is a powerful non-interactive stream editor used to perform basic text transformations, searches, substitutions, and deletions on an input stream (a file or pipelined input).

---

## Overview

Unlike interactive visual editors like `vim` or `nano`, `sed` processes incoming text line-by-line using a standardized cycle:

1. **Read** a line from standard input or file into the working pattern space.
2. **Execute** the specified commands sequentially against the pattern space.
3. **Output** the modified line to standard output (by default) and repeat until EOF.

---

## Basic Syntax

```bash
sed [options] 'command' file.txt
```

### Common Flags & Options

| Option | Description |
|---|---|
| `-i` | Edit files in-place (overwrites original file) |
| `-i.bak` | Edit in-place while keeping an untouched backup copy (`file.txt.bak`) |
| `-n` | Suppress automatic printing of the pattern space |
| `-e` | Specify multiple editing commands to execute in a single invocation |
| `-E` or `-r` | Use Extended Regular Expressions (ERE) instead of Basic (BRE) |

---

## 1. Find & Replace (Substitution)

The `s` command substitutes matched regular expressions with replacement strings.

```bash
sed 's/pattern/replacement/flags' file.txt
```

### Basic Replacements

```bash
# Replace first occurrence of 'apple' with 'orange' on every line
sed 's/apple/orange/' fruits.txt

# Replace all occurrences (globally) on every line
sed 's/apple/orange/g' fruits.txt

# Case-insensitive matching ('i' flag)
sed 's/apple/orange/gi' fruits.txt

# Replace only the 2nd occurrence on each line
sed 's/apple/orange/2' fruits.txt
```

### In-Place Editing (Saving Directly to File)

```bash
# Edit file directly in-place
sed -i 's/foo/bar/g' config.conf

# Edit in-place with an automatic backup file created
sed -i.bak 's/foo/bar/g' config.conf
```

### Custom Delimiters

When substituting file paths or URLs, avoid "leaning toothpick syndrome" (escaping slashes like `\/`) by switching to an alternate delimiter:

```bash
# Using '#' as a delimiter
sed 's#/usr/local/bin#/usr/bin#g' script.sh

# Using '|' as a delimiter
sed 's|https://old.domain.com|https://new.domain.com|g' urls.txt
```

---

## 2. Line Addressing & Range Matching

You can scope any `sed` operation to specific line numbers or patterns.

```bash
# Replace 'foo' with 'bar' ONLY on line 5
sed '5 s/foo/bar/' file.txt

# Replace 'foo' with 'bar' between lines 10 and 20
sed '10,20 s/foo/bar/g' file.txt

# Replace from line 5 to the end of the file ($)
sed '5,$ s/foo/bar/g' file.txt

# Replace only on lines matching the string 'prod'
sed '/prod/ s/debug/error/g' app.log

# Replace within a range bounded by two patterns
sed '/START/,/END/ s/foo/bar/g' file.txt
```

---

## 3. Deleting Lines (`d`)

```bash
# Delete line 3
sed '3d' file.txt

# Delete lines 5 through 10
sed '5,10d' file.txt

# Delete the last line
sed '$d' file.txt

# Delete all lines containing 'DEBUG'
sed '/DEBUG/d' app.log

# Delete lines that DO NOT match 'ERROR' (! negates the match)
sed '/ERROR/!d' app.log

# Delete all blank or empty lines
sed '/^$/d' file.txt

# Delete lines containing only spaces or tabs
sed '/^[[:blank:]]*$/d' file.txt
```

---

## 4. Printing Specific Lines (`p` with `-n`)

Combine the `-n` (silent) flag with `p` (print) to display only lines of interest:

```bash
# Print line 10 only
sed -n '10p' file.txt

# Print lines 15 through 25
sed -n '15,25p' file.txt

# Print only lines containing 'CRITICAL' (equivalent to grep)
sed -n '/CRITICAL/p' app.log

# Print lines bounded between two pattern matches
sed -n '/START/,/END/p' log.txt
```

---

## 5. Inserting, Appending & Changing Lines

| Command | Action |
|---|---|
| `i` | Insert text **before** the targeted line |
| `a` | Append text **after** the targeted line |
| `c` | Change/replace the entire matched line |

```bash
# Insert a comment before line 1 (prepends a file header)
sed '1i # Configuration Start' config.ini

# Append text after the line containing '[settings]'
sed '/\[settings\]/a port=8080' config.ini

# Replace entire line 4 with new content
sed '4c timeout=30' config.ini
```

---

## 6. Regex & Capture Groups (Backreferences)

Use the `-E` flag with `\1`, `\2`, etc., to reference capture groups in replacements:

```bash
# Swap first and last names (e.g., "John Doe" -> "Doe, John")
sed -E 's/([A-Za-z]+) ([A-Za-z]+)/\2, \1/' names.txt

# Comment out lines starting with 'Listen'
sed -i -E 's/^(Listen [0-9]+)/# \1/' httpd.conf

# Strip leading whitespace from all lines
sed 's/^[ \t]*//' file.txt

# Strip trailing whitespace from all lines
sed 's/[ \t]*$//' file.txt
```

---

## 7. Chaining Multiple Commands

Run multiple edits in a single execution using `-e` or semicolons (`;`):

```bash
# Using multiple -e expressions
sed -e 's/alpha/beta/g' -e 's/foo/bar/g' file.txt

# Using semicolons within a single command string
sed 's/alpha/beta/g; 5d; 10,20p' file.txt
```

---

## Quick Reference / Cheat Sheet

| Task | Command |
|---|---|
| Substitute all occurrences | `sed 's/find/replace/g' file` |
| Edit file in-place | `sed -i 's/find/replace/g' file` |
| Delete blank lines | `sed '/^$/d' file` |
| Delete line $N$ | `sed 'Nd' file` |
| Print line $N$ only | `sed -n 'Np' file` |
| Print range of lines | `sed -n '10,20p' file` |
| Delete lines matching regex | `sed '/pattern/d' file` |
| Alternate delimiter (paths) | `sed 's#http://#https://#g' file` |
| Strip leading spaces | `sed 's/^[ \t]*//' file` |

---

## Key Principles & Notes

- **Safe Defaults:** Without the `-i` flag, `sed` never alters original files; it outputs the modified stream to `stdout`.
- **macOS vs. Linux In-Place Difference:**
  - **Linux (GNU `sed`):** Accepts bare `-i` (e.g., `sed -i 's/a/b/' file.txt`).
  - **macOS (BSD `sed`):** Requires an explicit backup suffix argument, even if empty (e.g., `sed -i '' 's/a/b/' file.txt`).
- **Flexible Delimiters:** Any character following `s` serves as the pattern boundary (`s/`, `s#`, `s|`, `s:`).