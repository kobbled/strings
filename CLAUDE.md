# CLAUDE.md — lib/Strings

## Purpose

`Strings` is the Layer 1 string-utility library for Ka-Boost. Karel's standard library has almost no string manipulation beyond `SUB_STR`, `STR_LEN`, `CNV_INT_STR`, and `CNV_REAL_STR`. This module fills every gap: splitting, trimming, type conversion (both directions), path parsing, validation, register-to-string formatting, and binary encoding. It is a direct dependency of `errors`, `csv`, `display`, `files`, `KUnit`, `draw`, and virtually every module above Layer 1.

---

## Repository Layout

```
lib/Strings/
├── package.json              # rossum manifest — name "Strings", version 0.0.2
├── README.md                 # legacy minimal readme (no API docs)
├── include/
│   ├── strings.klh           # public header — 52 ROUTINE declarations
│   ├── strings.klt           # type definitions: SCELLS, STRING2D
│   └── strings.private.klh   # private header (empty — no private API)
├── src/
│   └── strings.kl            # full implementation, 1272 lines
├── test/
│   └── test_strings.kl       # KUnit test suite, 501 lines, 33 tests
└── tp/
    └── str_i_to_s.kl         # TP-callable interface: AR[1]→i, AR[2]→SR[n]
```

---

## Full API Reference

### Types (`strings.klt`)

```
SCELLS   = ARRAY[100] OF STRING[10]
STRING2D = ARRAY[100] OF SCELLS
```

These 2D string array types are available globally when `strings.klt` is included. `STRING2D` is rarely used directly but is available for CSV-like data manipulation.

---

### Splitting & Parsing

```
split_str(str:STRING; sep:STRING; out:ARRAY[*] OF STRING)
```
Splits `str` on single-char (or short-string) separator `sep` into `out`. Fills unused slots with `''`. Stops filling when the array is full and stores the remainder in the last slot. Modifies `out` in place; no return value.

```
extract_str(str:STRING; sep:STRING; start_idx, stop_idx:INTEGER) : STRING
```
Returns the substring between the `start_idx`-th and `stop_idx`-th occurrence of `sep`. 1-indexed. If `start_idx == stop_idx`, returns a single token. Avoids the need to pre-declare an output array — use when only one or a range of tokens are needed.

```
rev_split(str:STRING; sep:STRING; out:ARRAY[*] OF STRING)
```
Like `split_str` but scans right-to-left. `out[1]` = rightmost token, `out[2]` = second-from-right, etc. Used by `splitext`, `get_ext`, `basename` internally.

```
char_count(str:STRING; char:STRING) : INTEGER
```
Counts non-overlapping occurrences of `char` in `str`. Returns 0 if not found.

```
char_index(str:STRING; char:STRING) : INTEGER
```
Returns 1-based index of first occurrence of `char` in `str`. Returns 0 if not found.

```
takeStr(str:STRING; char:STRING) : STRING
```
Returns everything before the first occurrence of `char`. Returns uninitialized if `char` not found.

```
takeNextStr(str:STRING; char:STRING) : STRING
```
Returns everything after the first occurrence of `char`. Returns uninitialized if `char` not found.

```
getIntInStr(str:STRING) : INTEGER
```
Scans `str` left-to-right and returns the first run of consecutive digit characters as an INTEGER. Returns 0 if no digits found. Useful for extracting register numbers from labels like `'R[15]'`.

---

### Trimming & Formatting

```
lstrip(s:STRING) : STRING
rstrip(s:STRING) : STRING
```
Strip leading/trailing spaces respectively. Do not strip other whitespace (tabs, CR).

```
lstripChar(s:STRING; char:STRING) : STRING
rstripChar(s:STRING; char:STRING) : STRING
```
Strip leading/trailing occurrences of a specific single character.

```
delim_strip(s:STRING; delim:STRING) : STRING
```
Strips trailing occurrences of `delim` from right side of `s`. Scans right-to-left and stops at first non-delim character.

```
to_upper(str:STRING) : STRING
```
ASCII uppercase conversion. Converts a–z (ord 97–122) to A–Z by subtracting 32. Leaves all other characters unchanged.

```
str_parse(str:STRING; max_length:INTEGER) : STRING
```
Inserts `chr(13)` (CR) every `max_length` characters, resetting the counter at any existing CR. Used to word-wrap long messages for the FANUC TP display (40-char line limit).

```
remove_char(str:STRING; char:STRING) : STRING
```
Returns a copy of `str` with all occurrences of `char` removed. Uses rolling-window approach. Output limited to 64 characters.

```
esc_quotes(s:STRING) : STRING
```
Prepends `\` before each `'` (chr(44)) or `"` character. Used for JSON/CSV escaping.

```
delim_check(d:STRING)
```
Sanity-checks that `d` is a single recognized delimiter character (space, colon, semicolon, comma, slash). Writes to `TPERROR` if invalid. Does not abort — caller decides how to handle.

---

### Path Utilities

```
splitext(s:STRING) : STRING
```
Returns filename without extension. Internally calls `rev_split` on `'.'`; returns `split_arr[2]` (everything before the last dot). Note: confusingly named — returns stem, not extension.

```
get_ext(s:STRING) : STRING
```
Returns file extension (after last `'.'`). Returns `split_arr[1]` from `rev_split`.

```
get_device(s:STRING) : STRING
```
Returns device prefix from a FANUC path. Tries `':\\'` split first, then `':'` split. Returns part before the colon (e.g., `'UD1'` from `'UD1:\file.kl'`).

```
basename(s:STRING) : STRING
```
Returns filename component only. Tries `'\'` split, then `':'` split. Returns `split_arr[1]` from `rev_split`.

```
get_progname(s:STRING) : STRING
```
Extracts program name from FANUC variable reference format `'[progname]varname'`. Returns `''` if brackets not found or if `[` is followed by a digit (array index like `arr[1]`).

```
get_varname(s:STRING) : STRING
```
Extracts variable name from `'[progname]varname'` format. Returns the entire string if no brackets found (i.e., no program prefix).

---

### Type Conversions — Scalar to String

```
b_to_s(b:BOOLEAN) : STRING
```
Returns `'true'` or `'false'`. No variation.

```
i_to_s(i:INTEGER) : STRING
```
Converts integer to string using `CNV_INT_STR`, then `lstrip`s leading spaces. Returns `'null'` if `UNINIT(i)`.

```
r_to_s(r:REAL) : STRING
```
Converts real to string with exactly 3 decimal places via `CNV_REAL_STR`, then:
- Strips leading spaces
- Fixes `-.234` → `-0.234`
- Fixes `.234` → `0.234`
- Strips trailing zeros (keeps at least one decimal place, e.g. `1.0` not `1.`)
Returns `'null'` if `UNINIT(r)`.

```
p_to_s(p:XYZWPR) : STRING
```
Returns two-line human-readable format with labels:
```
X: 1.1 Y: 2.2 Z: 3.3
W: 4.4 P: 5.5 R: 6.6
```

```
pose_to_s(p:XYZWPR; delim:STRING) : STRING
```
Returns `x,y,z,w,p,r` as delimited string (6 values). Calls `delim_check` first.

```
posecfg2str(p:XYZWPR; grp_no:INTEGER) : STRING
```
Compact pose string including CONFIG data. Format: `'GP1: FUT,0,0,0 :\n x,y,z,w,p,r'`. Uses `CNV_CONF_STR` and removes spaces from config string.

```
vec_to_s(v:VECTOR; delim:STRING) : STRING
```
Returns `x,y,z` as delimited string (3 values). Calls `delim_check`.

```
joint_to_s(pose:JOINTPOS; delim:STRING) : STRING
```
Converts `JOINTPOS` to delimited real string. Uses `CNV_JPOS_REL` to get 9-element real array; skips uninitialized joints.

```
p_to_s_conf(p:XYZWPREXT; grp_no:INTEGER; uframe_no:INTEGER; utool_no:INTEGER) : STRING
```
Full extended pose string including group, frame, tool, config, and labeled XYZ/WPR with units (mm, deg). Multi-line. Writes to `TPERROR` and returns partial string if frame numbers are uninitialized.

```
rarr_to_s(arr:ARRAY[*] OF REAL; delim:STRING) : STRING
```
Joins real array into delimited string. Skips uninitialized elements. Output max 128 chars.

```
iarr_to_s(arr:ARRAY[*] OF INTEGER; delim:STRING) : STRING
```
Same as `rarr_to_s` but for INTEGER arrays.

```
sarr_to_s(arr:ARRAY[*] OF STRING; delim:STRING) : STRING
```
Same but for STRING arrays. Skips both uninitialized and empty-string elements.

```
rarr2d_to_s(arr:ARRAY[*,*] OF REAL; row:ARRAY[*] OF REAL; row_delim, col_delim:STRING) : STRING
```
Serializes 2D real array. Each row is joined with `row_delim`; rows are joined with `col_delim`. Requires a scratch `row` array of matching column size.

---

### Type Conversions — String to Scalar

```
s_to_i(s:STRING) : INTEGER
```
Converts string to integer. Handles negative numbers. Uses `CNV_STR_INT` internally. Returns 0 on non-numeric input.

```
s_to_r(s:STRING) : REAL
```
Converts string to real. Uses `CNV_STR_REAL`.

```
s_to_b(s:STRING) : BOOLEAN
```
Returns `TRUE` if `s == 'true'`, `FALSE` otherwise.

```
s_to_config(conf_str:STRING) : CONFIG
```
Converts config string (as produced by `CNV_CONF_STR`) back to a `CONFIG` type using `CNV_STR_CONF`.

```
s_to_vec(s:STRING; delim:STRING) : VECTOR
```
Parses `x,y,z` delimited string into `VECTOR`. Uses `extract_str` for each component.

```
s_to_xyzwpr(s:STRING; delim:STRING) : XYZWPR
```
Parses `x,y,z,w,p,r` delimited string into `XYZWPR`.

```
s_to_joint(s:STRING; delim:STRING) : JOINTPOS
```
Parses delimited real string into `JOINTPOS`. Uses `s_to_arr` → `sarr_to_rarr` → sets uninitialized to `0.0` → `CNV_REL_JPOS`.

---

### Type Conversions — String to Array

```
s_to_arr(s:STRING; delim:STRING; out_arr:ARRAY[*] OF STRING)
```
Splits delimited string into STRING array. Handles strings longer than 127 chars via chunked reading (reads characters `delim`-by-`delim` to work around Karel string length limits). This is the only routine in the module that handles very long strings specially.

```
s_to_rarr(s:STRING; delim:STRING; out_arr:ARRAY[*] OF REAL)
```
Splits delimited string into REAL array. Calls `s_to_arr` then converts each element.

```
s_to_iarr(s:STRING; delim:STRING; out_arr:ARRAY[*] OF INTEGER)
```
Splits delimited string into INTEGER array.

---

### Array-to-Array Conversions

```
sarr_to_rarr(sarr:ARRAY[*] OF STRING; out_rarr:ARRAY[*] OF REAL)
rarr_to_sarr(rarr:ARRAY[*] OF REAL; out_sarr:ARRAY[*] OF STRING)
sarr_to_iarr(sarr:ARRAY[*] OF STRING; out_iarr:ARRAY[*] OF INTEGER)
```
Element-wise type conversions between 1D arrays. Used internally by `s_to_rarr`, `s_to_iarr`, and `s_to_joint`.

---

### Register-to-String

```
numreg_to_s(reg_no:INTEGER) : STRING
```
Reads `R[reg_no]` and returns its value as a string (via `r_to_s`).

```
strreg_to_s(reg_no:INTEGER) : STRING
```
Reads `SR[reg_no]` and returns its string value directly.

```
posreg_to_s(reg_no, grp_no:INTEGER) : STRING
```
Reads `PR[reg_no]` and returns formatted string. Uses `p_to_s_conf` for Cartesian poses or `joint_to_s` for joint poses.

---

### Validation

```
charisnum(s:STRING) : BOOLEAN
```
Returns `TRUE` if `s` is a single digit character ('0'–'9'). Writes to `TPERROR` and returns `FALSE` if `STR_LEN(s) > 1`.

```
strisint(s:STRING) : BOOLEAN
```
Returns `TRUE` if `s` represents a valid integer (optional leading `-`, all remaining chars digits).

```
strisreal(s:STRING) : BOOLEAN
```
Returns `TRUE` if `s` represents a valid real number. Handles: optional leading `-`, one optional `.`, optional scientific notation `E+dd` or `E-dd`. Strips whitespace first.

```
search_str(srchStr:STRING; str:STRING) : BOOLEAN
```
Returns `TRUE` if `srchStr` is found anywhere within `str`. Linear scan.

---

### Binary Encoding

```
bin_to_i(s:STRING) : INTEGER
```
Converts binary string (e.g., `'10110'`) to integer. MSB-first interpretation. Ignores non-`'0'`/`'1'` characters.

```
i_to_byte(i:INTEGER; msb:BOOLEAN) : STRING
```
Converts integer to binary string. If `msb=TRUE`, MSB is first; if `msb=FALSE`, LSB is first. Always returns a full byte (8 characters with leading zeros).

---

### TP Interface (`tp/str_i_to_s.kl`)

Exposes `i_to_s` to TP programs via `AR[]` argument passing:
- `AR[1]` = integer value to convert
- `AR[2]` = string register number to write result into

Registered in `package.json` under `tp-interfaces` as program name `str_i_to_s`.

---

## Core Patterns

### Pattern 1: Splitting CSV/delimited data

Used by `csv`, `files`, and virtually all serialization code:

```karel
%from strings.klh %import split_str, extract_str, char_count

-- Fixed number of fields — use split_str
VAR tokens : ARRAY[6] OF STRING[16]
split_str('10.0,20.0,30.0', ',', tokens)
-- tokens[1]='10.0', tokens[2]='20.0', tokens[3]='30.0'

-- Random access to a single field — use extract_str
VAR y_str : STRING[16]
y_str = extract_str('10.0,20.0,30.0', ',', 2, 2)
-- y_str = '20.0'

-- Range extraction
VAR xyz_str : STRING[48]
xyz_str = extract_str('10.0,20.0,30.0,-90.0,0.0,180.0', ',', 1, 3)
-- xyz_str = '10.0,20.0,30.0'

-- Counting fields before splitting
VAR count : INTEGER
count = char_count(csv_line, ',')  -- used by csv.kl before read_type
```

### Pattern 2: Error message construction

Used throughout every module — `i_to_s` and `r_to_s` are the most-called routines in the codebase:

```karel
%from strings.klh %import i_to_s, r_to_s, vec_to_s
%from errors.klh %import karelError

-- Compose error messages with values
karelError(INVALID_INDEX, 'index ' + i_to_s(idx) + ' exceeds max ' + i_to_s(max_idx), ER_ABORT)

-- Debug logging with numeric data (draw.kl pattern)
usrdis__print(DEBUG, 'center:' + vec_to_s(origin, ','))
usrdis__print(DEBUG, 'idx:' + i_to_s(i) + ' det:' + r_to_s(det))
```

### Pattern 3: TP display line-wrapping

Used by `display.kl` for all TPDISPLAY output:

```karel
%from strings.klh %import str_parse
%define MAX_DISPLAY_LENGTH 40

WRITE TPDISPLAY(str_parse(long_message, MAX_DISPLAY_LENGTH), CR)
-- Automatically inserts CR every 40 characters
```

### Pattern 4: File path decomposition

Used by `files`, `csv`, and build tools to extract components from FANUC file paths:

```karel
%from strings.klh %import basename, get_device, splitext, get_ext

VAR path : STRING[32]
path = 'UD1:\subdir\myfile.kl'

basename(path)    -- 'myfile.kl'
get_device(path)  -- 'UD1'
splitext(path)    -- 'myfile'  (stem, not extension — see gotchas)
get_ext(path)     -- 'kl'
```

### Pattern 5: Pose/register-to-string for display

Used by `display.kl` and `KUnit` for test output and register inspection:

```karel
%from strings.klh %import posreg_to_s, numreg_to_s, strreg_to_s, i_to_s

-- Format register labels for display
RETURN('R[' + i_to_s(reg_no) + ']=' + numreg_to_s(reg_no))
RETURN('SR[' + i_to_s(reg_no) + ']=' + strreg_to_s(reg_no))
RETURN('PR[' + i_to_s(reg_no) + ']=' + posreg_to_s(reg_no, grp_no))
```

### Pattern 6: Round-trip serialization for CSV I/O

Used by `csv.kl` for reading/writing robot data to files:

```karel
%from strings.klh %import s_to_rarr, s_to_xyzwpr, pose_to_s, vec_to_s

-- Writing
csv__write_row_xyzwpr(my_pose)   -- internally uses pose_to_s

-- Reading back
VAR line : STRING[127]
VAR pose : XYZWPR
line = csv__read_line()
pose = s_to_xyzwpr(line, ',')
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `splitext` used to get extension | Returns `'myfile'` when you expected `'kl'` | Use `get_ext` for the extension; `splitext` returns the stem |
| `split_str` overflow — array smaller than field count | Last element contains all remaining tokens concatenated | Size the output array to at least `char_count(str, sep) + 1` |
| `extract_str` with `start_idx=1, stop_idx=1` on a string with no delimiter | Returns entire string instead of empty | Verify `char_count > 0` before calling `extract_str` |
| `delim_check` used as a validation guard | Code continues even when delimiter is invalid — `delim_check` only writes to TPERROR | `delim_check` is advisory only; add your own validation if you need to abort |
| `r_to_s` precision — comparing round-tripped reals | `r_to_s(1.23456)` → `'1.235'` (3 decimal places) — `s_to_r(r_to_s(x)) != x` | Use `r_to_s` only for display; for storage use 6+ decimal places with `CNV_REAL_STR` directly |
| `charisnum` called with multi-char string | Always returns FALSE; writes TPERROR | Only call with single character: `charisnum(SUB_STR(s, i, 1))` |
| `get_progname` / `get_varname` on array variable like `arr[1]` | Returns empty string for progname because `[1]` is treated as numeric bracket | This is intentional — the bracket detection checks if `[` is followed by a digit |
| `s_to_arr` with string > 127 chars | Works via internal chunking, but output strings are limited to `STRING[16]` elements | Maximum individual token length is 16 characters when using `s_to_arr` |
| `str_parse` with `max_length=40` but content has existing CRs | CRs reset the line length counter; short lines after CR are fine | Expected behavior — `str_parse` respects existing CRs |

---

## Dependencies

### What Strings depends on

| Module | Layer | Why |
|--------|-------|-----|
| `errors` | 1 | `karelError`, `CHK_STAT` for error posting |
| `systemlib` | 1 | Type definitions, `TPDISPLAY`, `TPERROR` macros |
| `TPElib` | 5 | `tpe__get_int_arg` in `tp/str_i_to_s.kl` |
| `ktransw-macros` | 0 | Build macros (header guards, namespace) |

Note: The circular-looking dependency on `TPElib` (Layer 5) is isolated to `tp/str_i_to_s.kl`, which is only compiled when building TP interfaces. The core `src/strings.kl` does not depend on `TPElib`.

### What depends on Strings (direct)

| Module | Usage |
|--------|-------|
| `errors` | `str_parse` for TP display line-wrapping of error messages |
| `csv` | `s_to_rarr`, `s_to_vec`, `s_to_xyzwpr`, `i_to_s`, `r_to_s`, `delim_strip`, `char_count` |
| `display` | `str_parse`, `i_to_s`, `strreg_to_s`, `numreg_to_s`, `posreg_to_s` |
| `files` | `extract_str`, `s_to_i`, `s_to_iarr`, `s_to_rarr`, `s_to_arr` |
| `KUnit` | `split_str`, `i_to_s`, `r_to_s`, `pose_to_s` |
| `draw/drawlib` | `i_to_s`, `r_to_s`, `vec_to_s`, `b_to_s` |
| `hash` | String-based key hashing utilities |
| `pose` | `r_to_s`, `i_to_s`, `p_to_s_conf`, `vec_to_s` |

All Layer 3–7 modules depend on Strings transitively through `csv`, `display`, or `files`.

---

## Build & Integration Notes

- `strings.kl` is the only compiled output (`"source": ["src/strings.kl"]` in `package.json`).
- The header `strings.klh` is included everywhere via `%from strings.klh %import ...` — never `%include strings.kl`.
- `strings.klt` (type definitions) is included separately only when `SCELLS` or `STRING2D` types are needed. Most modules do not need it.
- The TP interface `tp/str_i_to_s.kl` is compiled into a separate `.pc` file (`str_i_to_s.pc`) and called from TP programs as a subprogram. It is listed in `"tp-interfaces"` in `package.json` so rossum includes it in the build.
- `r_to_s` uses a hardcoded `%define DECIMAL_PLACES 3` inside the routine body. There is no way to configure precision from the call site.
- Karel `STRING` types have an implicit maximum length. Intermediate strings in `remove_char` and `esc_quotes` are limited to 64/128 characters respectively. Long strings will be silently truncated.
- `s_to_arr` handles strings longer than 127 characters through internal chunked processing — it is the one routine that works correctly with very long inputs.
