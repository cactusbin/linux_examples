# Text Processing: Removing Blank Lines & Regex Replacements

## Overview
Comprehensive examples for condensing multiple newlines and performing regex-based line replacements using sed, awk, and perl.

---

## Task 1: Condense Multiple Newlines to Single Newline

### sed (POSIX-compliant, portable)
```bash
# Most idiomatic: squeeze multiple blank lines into one
somecommand | sed '/^$/N;/^\n$/D'

# How it works:
# /^$/          - Match empty lines
# N             - Append next line to pattern space
# /^\n$/        - If pattern space is just a newline (two blank lines)
# D             - Delete first line of pattern space, restart
```

```bash
# Alternative: simpler but requires GNU sed
somecommand | sed '/^$/{ :a; N; /^\n*$/ba; }'

# How it works:
# /^$/          - Match empty line
# :a            - Label 'a' for branching
# N             - Append next line
# /^\n*$/       - If only newlines accumulated
# ba            - Branch back to label 'a'
```

```bash
# Most portable and simple (GNU sed)
somecommand | sed '/./,/^$/!d'

# How it works:
# /./           - Lines with any character
# ,/^$/         - Through next empty line
# !d            - Delete lines NOT in this range (consecutive blanks)
```

### awk (Highly readable, portable)
```bash
# Idiomatic awk - track consecutive blank lines
somecommand | awk 'NF {blank=0; print} !NF {if (!blank++) print}'

# How it works:
# NF            - Number of fields (>0 for non-empty lines)
# blank=0       - Reset blank counter on content
# !NF           - Empty line detected
# !blank++      - Only print first consecutive blank (post-increment)
```

```bash
# Alternative: using previous line tracking
somecommand | awk '!NF {if (prev) print; prev=0; next} {prev=1; print}'

# How it works:
# !NF           - Empty line
# prev          - Was previous line non-empty?
# next          - Skip to next line (don't set prev=1)
```

### perl (Most powerful, one-liner friendly)
```bash
# Idiomatic perl - paragraph mode
somecommand | perl -00 -pe ''

# How it works:
# -00           - Paragraph mode: treats blank lines as record separators
# -p            - Print each record automatically
# -e ''         - Empty expression (just enable -p mode)
# Result: Blank line sequences become single separator
```

```bash
# Explicit blank line tracking
somecommand | perl -ne 'print if /\S/ || !$blank++; $blank = 0 if /\S/'

# How it works:
# -n            - Loop without auto-print
# -e            - Execute inline code
# /\S/          - Non-whitespace (content line)
# $blank++      - Increment blank counter (false first time)
# $blank = 0    - Reset on content
```

---

## Task 2: Remove/Replace Lines with Regex (Including Capture Groups)

### sed (Stream editor - designed for this)
```bash
# Delete lines matching pattern
somecommand | sed '/pattern/d'

# Delete lines with extended regex (ERE)
somecommand | sed -E '/^(foo|bar)[0-9]+$/d'

# How it works:
# -E            - Extended regex (no backslash escaping for +, |, ())
# /pattern/     - Match pattern
# d             - Delete matching lines
```

```bash
# Replace entire line based on match
somecommand | sed -E '/error/c\REDACTED'

# How it works:
# /error/       - Find lines containing 'error'
# c\            - Change (replace) entire line
# REDACTED      - Replacement text
```

```bash
# Replace using capture groups
somecommand | sed -E 's/^(.*) ([0-9]+)$/\1 [\2]/'

# How it works:
# s/            - Substitute command
# ^(.*)         - Group 1: capture everything from start
# ([0-9]+)      - Group 2: capture digits
# $/            - Until end of line
# \1 [\2]       - Replace with: group1 [group2]
```

```bash
# Conditional replacement with address
somecommand | sed -E '/START/,/END/ s/(foo)/bar_\1_baz/'

# How it works:
# /START/,/END/ - Address range from START to END
# s/            - Only apply substitution in this range
# (foo)         - Capture 'foo'
# bar_\1_baz    - Wrap captured group
```

### awk (Better for field-based operations)
```bash
# Delete lines matching pattern
somecommand | awk '!/pattern/'

# How it works:
# !/pattern/    - Negation: print lines NOT matching
# Default action is print when condition true
```

```bash
# Replace with regex capture (gensub - GNU awk)
somecommand | awk '{print gensub(/^(.*) ([0-9]+)$/, "\\1 [\\2]", 1)}'

# How it works:
# gensub()      - GNU awk's regex substitution function
# (.*) ([0-9]+) - Two capture groups
# \\1 [\\2]     - Reference groups (double backslash in awk strings)
# 1             - Replace first occurrence
```

```bash
# Replace using match() and substr (POSIX awk)
somecommand | awk 'match($0, /^(.*) ([0-9]+)$/, a) {print a[1] " [" a[2] "]"}'

# How it works:
# match()       - Find pattern, store groups in array 'a'
# $0            - Entire line
# a[1], a[2]    - Captured groups
```

```bash
# Conditional modification
somecommand | awk '/pattern/ {gsub(/foo/, "bar")} 1'

# How it works:
# /pattern/     - Condition: line matches pattern
# gsub()        - Global substitution on $0
# 1             - Always true: print all lines (modified or not)
```

### perl (Most flexible for complex regex)
```bash
# Delete lines matching pattern
somecommand | perl -ne 'print unless /pattern/'

# How it works:
# -n            - Loop through lines
# -e            - Execute code
# unless        - Negative condition
```

```bash
# Replace with capture groups (most readable)
somecommand | perl -pe 's/^(.*) (\d+)$/$1 [$2]/'

# How it works:
# -p            - Auto-print mode
# s///          - Substitution
# (.*) (\d+)    - Two capture groups
# $1 [$2]       - Reference groups (perl style)
```

```bash
# Complex replacement with named captures
somecommand | perl -pe 's/(?<text>.*) (?<num>\d+)$/$+{text} [$+{num}]/'

# How it works:
# (?<text>...)  - Named capture group
# $+{name}      - Reference named group
# More readable for complex patterns
```

```bash
# Conditional replacement with lookahead/lookbehind
somecommand | perl -pe 's/(?<=ERROR: )(\w+)/***$1***/g'

# How it works:
# (?<=ERROR: )  - Positive lookbehind (match after 'ERROR: ')
# (\w+)         - Capture word
# ***$1***      - Wrap captured text
# g             - Global (all occurrences)
```

```bash
# Multi-line regex with capture
somecommand | perl -0777 -pe 's/(START.*?)END/$1MODIFIED/gs'

# How it works:
# -0777         - Slurp entire file into $_ 
# (START.*?)    - Non-greedy capture from START
# END           - Until END
# s             - Dot matches newlines
# g             - Global
```

---

## Recommendation Summary

### For Removing Redundant Newlines:
**Best choice: `perl -00 -pe ''`**
- Shortest, clearest intent
- Portable across all systems with perl
- Paragraph mode designed for this

**Alternative: `awk 'NF {blank=0; print} !NF {if (!blank++) print}'`**
- No perl required
- Very readable logic
- Portable POSIX awk

### For Regex Line Replacement:
**Best choice: `perl -pe 's/pattern/replacement/'`**
- Most powerful regex engine
- Clean capture group syntax ($1, $2, or named)
- Lookahead/lookbehind support
- One-liner friendly

**Alternative: `sed -E 's/pattern/replacement/'`**
- When perl unavailable
- Simpler patterns without advanced features
- Very portable (POSIX)

---

## Combined Examples

### Remove blank lines AND replace patterns
```bash
# Using perl (most concise)
somecommand | perl -00 -pe 's/(ERROR\s+)(\d+)/$1***$2***/g'

# Using sed + awk chain
somecommand | awk 'NF {blank=0; print} !NF {if (!blank++) print}' | sed -E 's/(ERROR[[:space:]]+)([0-9]+)/\1***\2***/g'
```

### Process log files
```bash
# Remove blank lines and anonymize IP addresses
cat log.txt | perl -00 -pe 's/(\d{1,3}\.\d{1,3}\.\d{1,3}\.)\d{1,3}/${1}XXX/g'

# How it works:
# -00           - Paragraph mode (removes extra blanks)
# (\d{1,3}\.)   - Capture first 3 octets with dots
# \d{1,3}       - Last octet (not captured)
# ${1}XXX       - Keep first 3, replace last with XXX
```

### Clean up command output
```bash
# Remove blank lines and extract version numbers
some-verbose-command 2>&1 | perl -ne 'print if /\S/; next if /^$/; s/Version:? (\d+\.\d+\.\d+)/v$1/ and print'

# How it works:
# /\S/          - Has non-whitespace
# next if /^$/  - Skip blank lines
# s///          - Substitute and test
# and print     - Print if substitution succeeded
```

---

## Performance Notes

- **sed**: Fastest for simple patterns, streams efficiently
- **awk**: Fast, better for field processing, good balance
- **perl**: Slightly slower, but negligible for most use cases; most powerful
- **All are streaming**: Process line-by-line (except perl -0777)

## Portability

- **Most portable**: POSIX sed and awk
- **GNU extensions**: sed -E, awk gensub() (widely available)
- **perl**: Nearly universal on Unix/Linux, not always on minimal containers
- **fish/bash/sh/dash**: All examples work in any shell (commands are external)
