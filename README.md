# Strings

String utility library for FANUC Karel — splitting, trimming, type conversion, path parsing, validation, and register formatting.

## Overview

Karel's standard library provides `SUB_STR`, `STR_LEN`, `CNV_INT_STR`, `CNV_REAL_STR`, and not much else. `Strings` fills every gap you'll hit when building robot programs that read files, format error messages, parse CSV data, or display values on the teach pendant. It is a direct dependency of `errors`, `csv`, `display`, `files`, and `KUnit`, and is transitively used by every module above Layer 1.

---

## Files

| File | Purpose |
|------|---------|
| `include/strings.klh` | Public header — declare-import this in any module that needs string utilities |
| `include/strings.klt` | Type definitions: `SCELLS`, `STRING2D` — only include when you need 2D string arrays |
| `include/strings.private.klh` | Empty private header (no private API) |
| `src/strings.kl` | Full implementation (1272 lines, 52 routines) |
| `test/test_strings.kl` | KUnit test suite (501 lines, 33 tests covering all major routines) |
| `tp/str_i_to_s.kl` | TP-callable wrapper: converts `AR[1]` integer to string, stores in `SR[AR[2]]` |
| `package.json` | rossum manifest — name `"Strings"`, version `0.0.2` |

---

## API Reference

Import only what you need:

```karel
%from strings.klh %import i_to_s, r_to_s, split_str
```

---

### Splitting & Parsing

#### `split_str(str, sep, out[])`
Splits `str` on separator `sep` into output array. Unused slots are set to `''`. If the array fills up before the string ends, the remainder goes into the last slot.

```karel
VAR tokens : ARRAY[6] OF STRING[16]
split_str('a,b,c,d', ',', tokens)
-- tokens[1]='a', tokens[2]='b', tokens[3]='c', tokens[4]='d', tokens[5..6]=''

-- Size the array to char_count + 1 to avoid overflow:
VAR count : INTEGER
count = char_count(line, ',') + 1
-- then declare your array accordingly
```

#### `extract_str(str, sep, start_idx, stop_idx) : STRING`
Returns the substring between the `start_idx`-th and `stop_idx`-th occurrence of `sep`. 1-indexed. More efficient than `split_str` when you only need one or two fields from a long string.

```karel
-- Get the 2nd token (y coordinate from 'x,y,z')
y_str = extract_str('10.5,20.0,30.0', ',', 2, 2)  -- '20.0'

-- Get tokens 1 through 3 as a substring
xyz = extract_str('10.5,20.0,30.0,-90.0,0.0,0.0', ',', 1, 3)  -- '10.5,20.0,30.0'
```

#### `rev_split(str, sep, out[])`
Splits right-to-left. `out[1]` = rightmost token. Used internally by `splitext`, `get_ext`, `basename`.

#### `char_count(str, char) : INTEGER`
Counts occurrences of `char` in `str`. Returns 0 if not found.

```karel
count = char_count('a,b,c', ',')  -- 2
```

#### `char_index(str, char) : INTEGER`
Returns 1-based index of first occurrence of `char`. Returns 0 if not found.

#### `takeStr(str, char) : STRING`
Returns everything before the first occurrence of `char`.

```karel
takeStr('myfile.kl', '.')  -- 'myfile'
```

#### `takeNextStr(str, char) : STRING`
Returns everything after the first occurrence of `char`.

```karel
takeNextStr('key=value', '=')  -- 'value'
```

#### `getIntInStr(str) : INTEGER`
Scans left-to-right and returns the first run of digit characters as an integer.

```karel
getIntInStr('R[15]')   -- 15
getIntInStr('PR[3]')   -- 3
getIntInStr('no nums') -- 0
```

---

### Trimming & Formatting

#### `lstrip(s) / rstrip(s) : STRING`
Strip leading/trailing spaces. Used internally by `i_to_s` and `r_to_s`.

#### `lstripChar(s, char) / rstripChar(s, char) : STRING`
Strip leading/trailing occurrences of a specific character.

```karel
lstripChar('000123', '0')  -- '123'
rstripChar('1.2300', '0')  -- '1.23'
```

#### `delim_strip(s, delim) : STRING`
Strips trailing occurrences of `delim` from the right side of `s`.

#### `to_upper(str) : STRING`
ASCII uppercase. Converts a-z; leaves everything else unchanged.

#### `str_parse(str, max_length) : STRING`
Inserts `chr(13)` every `max_length` characters. Use this before writing to `TPDISPLAY` to prevent line overflow. The FANUC TP display is 40 characters wide.

```karel
WRITE TPDISPLAY(str_parse(msg, 40), CR)
```

#### `remove_char(str, char) : STRING`
Returns a copy of `str` with all occurrences of `char` removed. Output limited to 64 characters.

```karel
remove_char('hello world', ' ')  -- 'helloworld'
```

#### `esc_quotes(s) : STRING`
Prepends `\` before each `'` or `"`. Used when writing strings that will be embedded in quoted contexts.

#### `delim_check(d)`
Advisory sanity check — writes to `TPERROR` if `d` is not a recognized delimiter (space, `:`, `;`, `,`, `/`). Does not abort.

---

### Path Utilities

```karel
basename('UD1:\sub\file.kl')    -- 'file.kl'
get_device('UD1:\sub\file.kl')  -- 'UD1'
splitext('file.kl')             -- 'file'   NOTE: returns STEM, not extension
get_ext('file.kl')              -- 'kl'
```

> **Warning:** `splitext` returns the filename stem (without extension), not the extension. This is the opposite of what Python's `os.path.splitext` returns. Use `get_ext` for the extension.

#### `get_progname(s) / get_varname(s) : STRING`
Parse FANUC variable reference format `'[progname]varname'`:

```karel
get_progname('[mylib]my_var')  -- 'mylib'
get_varname('[mylib]my_var')   -- 'my_var'
get_varname('plain_var')       -- 'plain_var'  (no brackets = returns as-is)
```

---

### Type Conversions — To String

#### `b_to_s(b) : STRING`
`TRUE` -> `'true'`, `FALSE` -> `'false'`.

#### `i_to_s(i) : STRING`
Integer to string, leading spaces stripped. Returns `'null'` if uninitialized.

```karel
i_to_s(42)    -- '42'
i_to_s(-7)    -- '-7'
```

#### `r_to_s(r) : STRING`
Real to string with 3 decimal places, trailing zeros stripped. Handles edge cases like `-.234` -> `'-0.234'`. Returns `'null'` if uninitialized.

```karel
r_to_s(3.14159)  -- '3.142'
r_to_s(1.0)      -- '1.0'   (keeps one decimal place)
r_to_s(-0.5)     -- '-0.5'
```

#### `p_to_s(p) : STRING`
Two-line labeled format:
```
X: 10.0 Y: 20.0 Z: 30.0
W: -90.0 P: 0.0 R: 180.0
```

#### `pose_to_s(p, delim) : STRING`
Six-value delimited: `'x,y,z,w,p,r'`

#### `posecfg2str(p, grp_no) : STRING`
Compact format including CONFIG: `'GP1: FUT,0,0,0 :\nx,y,z,w,p,r'`

#### `p_to_s_conf(p, grp_no, uframe_no, utool_no) : STRING`
Full extended pose with group, frame, tool, config labels, and units (mm/deg).

#### `vec_to_s(v, delim) : STRING`
`'x,y,z'` delimited.

#### `joint_to_s(pose, delim) : STRING`
Joint angles as delimited real string. Skips uninitialized axes.

#### `rarr_to_s(arr[], delim) : STRING`
Real array -> delimited string. Skips uninitialized elements.

#### `iarr_to_s(arr[], delim) : STRING`
Integer array -> delimited string.

#### `sarr_to_s(arr[], delim) : STRING`
String array -> delimited string. Skips empty and uninitialized elements.

#### `rarr2d_to_s(arr[,], row[], row_delim, col_delim) : STRING`
2D real array -> string. Rows separated by `col_delim`, values within rows by `row_delim`. Requires a scratch `row` array of matching column size.

---

### Type Conversions — From String

#### `s_to_i(s) : INTEGER`
Parses integer from string. Handles leading `-`.

#### `s_to_r(s) : REAL`
Parses real from string.

#### `s_to_b(s) : BOOLEAN`
`'true'` -> `TRUE`, anything else -> `FALSE`.

#### `s_to_config(conf_str) : CONFIG`
Converts config string (as produced by `CNV_CONF_STR`) back to `CONFIG` type.

#### `s_to_vec(s, delim) : VECTOR`
`'x,y,z'` delimited string -> `VECTOR`.

#### `s_to_xyzwpr(s, delim) : XYZWPR`
`'x,y,z,w,p,r'` delimited string -> `XYZWPR`.

#### `s_to_joint(s, delim) : JOINTPOS`
Delimited real string -> `JOINTPOS`. Uninitialized axes default to `0.0`.

#### `s_to_arr(s, delim, out_arr[])`
Delimited string -> STRING array. Handles strings longer than 127 characters via internal chunking. Individual token max length is 16 characters.

#### `s_to_rarr(s, delim, out_arr[])`
Delimited string -> REAL array.

#### `s_to_iarr(s, delim, out_arr[])`
Delimited string -> INTEGER array.

---

### Array-to-Array Conversions

```
sarr_to_rarr(sarr[], out_rarr[])  -- STRING[] to REAL[]
rarr_to_sarr(rarr[], out_sarr[])  -- REAL[] to STRING[]
sarr_to_iarr(sarr[], out_iarr[])  -- STRING[] to INTEGER[]
```

---

### Register-to-String

```
numreg_to_s(reg_no) : STRING          -- reads R[n], returns r_to_s value
strreg_to_s(reg_no) : STRING          -- reads SR[n], returns string value
posreg_to_s(reg_no, grp_no) : STRING  -- reads PR[n], returns formatted pose string
```

Used by `display.kl` to format register labels for the TP display.

---

### Validation

#### `charisnum(s) : BOOLEAN`
`TRUE` if `s` is a single digit character. Call with `SUB_STR(str, i, 1)`, not multi-char strings.

#### `strisint(s) : BOOLEAN`
`TRUE` if `s` is a valid integer (optional leading `-`, digits only).

#### `strisreal(s) : BOOLEAN`
`TRUE` if `s` is a valid real number. Handles scientific notation (`E+dd`/`E-dd`). Strips whitespace before checking.

#### `search_str(srchStr, str) : BOOLEAN`
`TRUE` if `srchStr` is a substring of `str`. Linear scan.

---

### Binary Encoding

#### `bin_to_i(s) : INTEGER`
Binary string -> integer. MSB-first. `'1010'` -> `10`.

#### `i_to_byte(i, msb) : STRING`
Integer -> 8-character binary string. `msb=TRUE` puts the most significant bit first.

```karel
i_to_byte(5, TRUE)   -- '00000101'
i_to_byte(5, FALSE)  -- '10100000'
```

---

## Common Patterns

### Pattern 1: Parsing a CSV line

```karel
%from strings.klh %import char_count, split_str, s_to_r

VAR
    line    : STRING[127]
    tokens  : ARRAY[12] OF STRING[16]
    x, y, z : REAL

-- After reading a line from a file:
IF char_count(line, ',') >= 2 THEN
    split_str(line, ',', tokens)
    x = s_to_r(tokens[1])
    y = s_to_r(tokens[2])
    z = s_to_r(tokens[3])
ENDIF
```

### Pattern 2: Error messages with values

```karel
%from strings.klh %import i_to_s, r_to_s
%from errors.klh %import karelError

karelError(INVALID_INDEX, &
    'Buffer index ' + i_to_s(idx) + ' exceeds max ' + i_to_s(buf_size), &
    ER_ABORT)
```

### Pattern 3: Round-trip pose serialization

```karel
%from strings.klh %import pose_to_s, s_to_xyzwpr

-- Write to CSV
csv_line = pose_to_s(my_pose, ',')

-- Read back
my_pose = s_to_xyzwpr(csv_line, ',')
```

### Pattern 4: TP display with line wrapping

```karel
%from strings.klh %import str_parse, i_to_s

WRITE TPDISPLAY(str_parse( &
    'Processing layer ' + i_to_s(layer) + ' of ' + i_to_s(total_layers), 40), CR)
```

### Pattern 5: File path decomposition

```karel
%from strings.klh %import basename, get_device, get_ext, splitext

VAR path : STRING[32]
path = 'UD1:\data\scan001.csv'

basename(path)   -- 'scan001.csv'
get_device(path) -- 'UD1'
splitext(path)   -- 'scan001'  (stem, NOT extension)
get_ext(path)    -- 'csv'
```

### Pattern 6: Validate input before conversion

```karel
%from strings.klh %import strisreal, s_to_r
%from errors.klh %import karelError

IF strisreal(user_input) THEN
    val = s_to_r(user_input)
ELSE
    karelError(VAR_UNINIT, 'Expected a real number, got: ' + user_input, ER_WARN)
ENDIF
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `splitext` used to get extension | Returns stem `'myfile'` instead of `'kl'` | Use `get_ext` for the extension; `splitext` returns filename without extension |
| `split_str` output array too small | Last element contains remaining unsplit tokens concatenated | Size array to `char_count(str, sep) + 1` |
| `extract_str` on string with no matching delimiter | Returns entire string | Check `char_count > 0` first |
| `delim_check` treated as a validator that stops execution | Code continues past invalid delimiters | `delim_check` only writes to TPERROR; add your own abort if needed |
| `r_to_s` used for persistent storage | Rounding to 3 decimal places loses precision | Use `r_to_s` for display only; use `CNV_REAL_STR` directly for storage |
| `charisnum` called with multi-char string | Always returns `FALSE` and writes TPERROR | Only pass single characters: `charisnum(SUB_STR(s, i, 1))` |
| `s_to_arr` with tokens longer than 16 chars | Tokens silently truncated | Individual token max is STRING[16]; for longer tokens use `split_str` |
| `i_to_byte(i, FALSE)` expected to return little-endian bytes | Returns bit-reversed string | `msb=FALSE` means LSB is placed at leftmost position in the 8-char string |

---

## Build Flow

`Strings` is Layer 1 in Ka-Boost. It compiles to a single `strings.pc` and (optionally) `str_i_to_s.pc` for the TP interface.

```shell
cd lib/Strings
mkdir build && cd build
rossum .. -w -o     # generates build.ninja
ninja               # strings.kl -> strings.pc
kpush               # upload to controller
```

See the [Ka-Boost readme](../../readme.md) for full build, deploy, and test instructions.
