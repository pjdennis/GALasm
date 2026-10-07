# GALasm Assembler: Behavioural Specification

Version 1.0 (October 2026). Describes the behaviour of GALasm 2.1 as maintained at
<https://github.com/pjdennis/GALasm>.

This document is under the MIT license (see `LICENSE` in this directory).

---

## 0. About this document

### 0.1 Purpose

This document specifies what a GALasm-compatible assembler does: the
command-line interface, the source language, how sources are turned into
fuse maps for four GAL devices, and the exact formats of the files it
writes.  It is written so that a programmer who has never seen the
GALasm source code can build a compatible assembler from it.

The accompanying test suite in `tests/` is the executable part of this
specification.  Where this document and the suite disagree, treat it as a
bug in one of them and resolve it deliberately.

### 0.2 Conventions

* **MUST**, **SHOULD**, and **MAY** have their usual RFC 2119 meanings.
* Sections marked *(non-normative)* explain or advise and impose no
  requirements.
* Pin numbers are physical DIP package pin numbers.
* A *fuse* has a value of 0 or 1 as written in the JEDEC file.
* `LF` is byte 0x0A, `CR` is 0x0D, `TAB` is 0x09, `STX` is 0x02, `ETX` is 0x03.
* Hexadecimal numbers are written `0x..`.
* `%2d`, `%3d` mean a decimal number right-aligned in 2 or 3 characters,
  padded with spaces.  `%04d` means at least 4 digits, padded with zeros.

### 0.3 Compatibility goal

For every input the reference accepts, a conforming implementation MUST
produce byte-identical `.jed`, `.fus`, `.pin` and `.chp` files, with three
exceptions:

* the values of the two program-identification lines in the JEDEC header
  (§9.2);
* the transmission checksum, which changes when those lines do (§9.5);
* native line endings on platforms whose text files use CR LF.

For every input the reference rejects, a conforming implementation MUST
also reject it.  It MUST report the same first error at the same line
(§8; the line is not specified for unexpected end of file), and MUST NOT
write any output file.

---

## 1. Supported devices

| Device | Pins | GND pin | VCC pin | Output cells (OLMCs) | Fuse-array rows × columns |
|---|---|---|---|---|---|
| GAL16V8 | 20 | 10 | 20 | 8, on pins 12–19 | 64 × 32 = 2048 |
| GAL20V8 | 24 | 12 | 24 | 8, on pins 15–22 | 64 × 40 = 2560 |
| GAL22V10 | 24 | 12 | 24 | 10, on pins 14–23 | 132 × 44 = 5808 |
| GAL20RA10 | 24 | 12 | 24 | 10, on pins 14–23 | 80 × 40 = 3200 |

An *OLMC pin* is a pin that has an output logic macrocell and so can be
driven by an equation.  Every other pin is an *input-only pin*.

---

## 2. Command line

```
galasm [options] <file>
```

### 2.1 Options

Options come before the file name.  Each is a `-` followed by one or more
option letters.  Letters may be combined: `-sfp` is the same as
`-s -f -p`.  Option letters are case-insensitive.

| Letter | Effect |
|---|---|
| `s` | Set the security fuse (`*G1` in the JEDEC file instead of `*G0`) |
| `c` | Do not write the `.chp` file |
| `f` | Do not write the `.fus` file |
| `p` | Do not write the `.pin` file |
| `a` | Omit `<STX>`, `<ETX>` and the transmission checksum from the JEDEC file (§9.6) |
| `w` | Write the JEDEC file with CR LF line endings on every platform |
| `v` | Verbose: also explain on the console why the GAL16V8/GAL20V8 mode was chosen (§10) |
| `h` or `?` | Print usage help and exit with status 0 without assembling anything |

A lone `-`, and an argument starting with `--`, end option processing.
The next argument is then taken as the file name.

These are usage errors:
* an unknown option letter;
* no file name;
* more than one file name.

On a usage error the implementation MUST print a usage message, MUST NOT
assemble anything, and MUST exit with a non-zero status (the reference
uses 5).

*(Non-normative.)* The reference also accepts some option groups written
without a leading `-` when more arguments follow (for example
`galasm sa file.pld`).  This accident does not need to be reproduced.

### 2.2 Output files

The output files go next to the input file.  Their names are the input
path with its extension replaced:

* If the file name contains a `.`, everything from the last `.` onwards is
  replaced by `.jed`, `.fus`, `.pin` or `.chp`.  So `my.design.pld` gives
  `my.design.jed`.
* Otherwise the extension is appended, so `design` gives `design.jed`.

The `.jed` file is always written.  The other three are written unless
suppressed by an option.  If assembly fails, no file is written.

*(Non-normative.)* The reference looks for the last `.` anywhere in the
argument, including in directory names, so `dir.v2/file` would give
`dir.jed`.  Implementations SHOULD only consider the final path
component.  The suite does not test paths with directories.

### 2.3 Exit status

| Situation | Status |
|---|---|
| Success, or `-h` | 0 |
| Usage error | non-zero (reference: 5) |
| Source error (§8) | non-zero (reference: 255) |
| Input file cannot be opened or read, or out of memory | non-zero (reference: 254) |

---

## 3. Source file structure

A source file has four parts, in this order:

1. **Line 1**: the device type (§3.2).
2. **Line 2**: the signature (§3.3).
3. **Pin declarations**: starting on line 3 (§4).
4. **Equations**: ending with the keyword `DESCRIPTION` (§5).

Everything after `DESCRIPTION` is ignored.

### 3.1 Characters, whitespace and comments

These rules apply to the pin declarations and the equations, not to
lines 1 and 2.

* **Whitespace.** Space, TAB and LF separate tokens.  Any byte outside
  the printable ASCII range 0x21–0x7E is also treated as whitespace.
  That includes CR and bytes ≥ 0x7F.
* **Comments.** A `;` starts a comment that runs to the end of the line.
* **Names** are made of the ASCII letters `A–Z`, `a–z` and the digits
  `0–9`.  Names are case-sensitive.
* **Operators:**

  | Meaning | Characters |
  |---|---|
  | Negation | `/` or `!` |
  | AND | `*` or `&` |
  | OR | `+` or `#` |
  | Assignment | `=` |
  | Suffix separator | `.` |

* **Line numbers** start at 1.  Every LF ends a line.

### 3.2 Line 1: device type

The file MUST begin, at its very first byte, with one of `GAL16V8`,
`GAL20V8`, `GAL22V10` or `GAL20RA10`.  The match is case-sensitive.  The
next byte MUST be a space, TAB or LF.  Otherwise the file is rejected
with error E1, reported at line 1.

Anything else on line 1 is ignored.

*(Non-normative.)* Because of this rule, the reference rejects files with
CR LF line endings on systems where text files are not translated
(Linux, macOS): the byte after the type is CR.  On Windows the C runtime
removes the CR and such files are accepted.  §12 recommends accepting
CR here.

### 3.3 Line 2: signature

The signature is taken from the bytes at the start of line 2, up to the
first LF or TAB or 8 bytes, whichever comes first.  Spaces and every
other byte are part of the signature, including `;`.  The rest of line 2
is ignored.

If the file ends before line 2 exists, report E2 (unexpected end of file).

Each signature byte gives 8 signature fuses, most significant bit first.
Byte *k* (0-based) gives signature fuses 8*k* … 8*k*+7.  Signature fuses
beyond the given bytes are 0.  There are always 64 signature fuses.

---

## 4. Pin declarations

Pin declarations start at the beginning of line 3.  There are exactly as
many pin names as the device has pins (20 or 24).  They are assigned to
pins 1, 2, 3, … in order.  Names may be spread over any number of lines,
separated by whitespace and comments.

### 4.1 Pin name syntax

A pin name is an optional negation sign (`/` or `!`) followed directly by
1–8 name characters.  The negation, if present, is stored as part of the
declared name; this matters for the output files, see §9.

The *base name* is the name without its negation sign.

### 4.2 Checks

Each name is checked when it is read, in the order below.  The first
failing check is reported, at the line where the name is.

| # | Check | Error |
|---|---|---|
| 1 | The first character must be a negation sign or a name character | E5 |
| 2 | A negation sign may only appear as the first character | E10 |
| 3 | A negation sign must be followed directly by a name character | E3 |
| 4 | At most 8 name characters (not counting the negation sign) | E4 |
| 5 | The base name must differ from the base names of all earlier pins, except earlier pins whose declared name is exactly `NC` | E9 |
| 6 | `GND` may only be declared on the GND pin | E6 |
| 7 | The GND pin must be declared exactly `GND` | E8 |
| 8 | `VCC` may only be declared on the VCC pin | E6 |
| 9 | The VCC pin must be declared exactly `VCC` | E7 |
| 10 | GAL22V10 only: the declared name must not be exactly `AR` or `SP` | E18 |

How a name ends affects which check fails:

* A name ends at the first character that is not a name character or a
  negation sign.
* If that character is something other than whitespace or a comment, for
  example `_` in `B_C`, the name before it (`B`) is accepted.  The `_`
  then fails check 1 as the start of the next name.
* A `/` inside a name (`B/C`) fails check 2.

If the file ends before all pins are declared, report E2.

`NC` (not connected) may be declared on any number of pins.  Pins named
`NC` cannot be used in equations.

*(Non-normative.)* Check 10 compares the declared name, so `/AR` is not
rejected in the reference.  It is harmless because such a pin could
never be used.

---

## 5. Equations

### 5.1 Grammar

```
equations   := equation { equation } "DESCRIPTION"
equation    := lhs "=" term { op term }
lhs         := [neg] target [ "." suffix ]
target      := pin-name | "AR" | "SP"           ; AR, SP: GAL22V10 only
term        := [neg] pin-name
op          := and | or
neg         := "/" | "!"
and         := "*" | "&"
or          := "+" | "#"
```

* Whitespace and comments may appear between any two tokens, except:
  * between a negation sign and the following name;
  * between `.` and the suffix letters.
* An equation ends when a term is not followed by an operator.  The next
  token then starts a new equation, or is `DESCRIPTION`.
* `DESCRIPTION` is recognised where an equation would start, if the text
  there begins with the 11 characters `DESCRIPTION`.  It is
  case-sensitive.
* If `DESCRIPTION` is the first token after the pin declarations, report
  E33 at that line.
* If the file ends before `DESCRIPTION`, report E2.

### 5.2 Semantics

* The right-hand side is a sum of products.  AND binds tighter than OR.
  There are no parentheses.
* Each maximal run of terms joined by AND is one *product term*.
* Every term refers to a pin by its base name: the name is looked up
  without the negation sign from its declaration.

**Polarity of a term.** A term is inverted if exactly one of these holds:
* the term is written with a negation sign;
* the referenced pin was declared with a negation sign.

For example, with pin 1 declared `/A`, the term `A` means "pin 1 low" and
`/A` means "pin 1 high".

**Polarity of an output.** Combine the negation on the left-hand side with
the negation in the target pin's declaration in the same way.  An
inverted result makes the output *active low*; otherwise it is *active
high*.

**VCC and GND.** These may appear as the only term of an equation (no
operator before or after them).
* `GND` makes that product term constantly false.
* `VCC` makes it constantly true.

### 5.3 Suffixes

The suffix letters are read up to the first non-letter.  More than 6
letters gives error E13.  The suffix is then identified as follows; any
other suffix is E13.

| Suffix | How it is recognised | Meaning | Allowed on |
|---|---|---|---|
| none | | Output; the type is chosen by the assembler (§6) | all |
| `.T` | any suffix whose first letter is `T` | Tristate output | all |
| `.R` | any suffix whose first letter is `R` | Registered output | all |
| `.E` | any suffix whose first letter is `E` | Output-enable product term for a `.T` or `.R` output | all |
| `.CLK` | exactly `CLK` | Clock product term of a registered output | GAL20RA10; elsewhere E34 |
| `.ARST` | exactly `ARST` | Asynchronous reset product term of a registered output | GAL20RA10; elsewhere E35 |
| `.APRST` | exactly `APRST` | Asynchronous preset product term of a registered output | GAL20RA10; elsewhere E36 |

Matching is case-sensitive, so `.t` is E13.  Because only the first letter
is checked for T, R and E, `.Tri`, `.Reg` and `.Enable` are accepted.

The order of checks when a `.` follows the target is:
1. GAL22V10 only: if the target is AR or SP, report E39.
2. Unknown suffix: E13.
3. Device restriction: E34, E35 or E36.

### 5.4 AR and SP (GAL22V10)

On the GAL22V10, `AR` (asynchronous reset) and `SP` (synchronous preset)
may be used as equation targets for the device-wide reset and preset
product terms.  The rules:

* They are recognised only where the name is not a declared pin name.
* They may not carry a suffix (E39).
* They may not be negated (E32).
* They may not be used as terms (E31).
* They may be defined at most once each (E40).
* Each is a single product term, so an OR gives E29.

---

## 6. Classifying outputs and choosing the mode

Before any fuse is computed, the whole equation section is read once to
classify every OLMC.  The fuse map depends on decisions that a later
equation can change, so this complete pass is required.

### 6.1 OLMC states

Each OLMC (plus AR and SP on the GAL22V10) starts *unused*.  The
equations are processed in file order.

**Target of an equation.** First the target must be valid:
* E11 if it is not a declared name (or AR/SP on the 22V10);
* E12 if it is `NC`;
* E32 if it is a negated AR or SP;
* E15 if it is a declared pin without an OLMC.

The equation then acts on the target OLMC according to its suffix:

| Suffix | Action |
|---|---|
| none, `.T`, `.R` | If the OLMC is *unused* or *input*, it becomes an output with the polarity from §5.2.  The kind is *undecided* (no suffix), *tristate* (`.T`) or *registered* (`.R`).  If it is already an output: E40 for AR/SP, E16 otherwise. |
| `.E` | In this order: inverted target polarity → E19; the OLMC already has an `.E` → E22; the OLMC is unused or input → E17; registered output on a GAL16V8/20V8 → E23; undecided output (defined without suffix) → E24.  Otherwise record that the OLMC has an enable equation. |
| `.CLK` / `.ARST` / `.APRST` | In this order: inverted target polarity → E19; the OLMC is unused → E42 / E43 / E44; this suffix already given for this OLMC → E45 / E46 / E47; the OLMC is not a registered output → E48.  Otherwise record it.  (An OLMC in the *input* state passes the "unused" check and then fails with E48.) |

**Terms of an equation.** Each term must be valid:
* E11 if it is not a declared pin name;
* E31 if it is AR or SP (22V10);
* E12 if it is `NC`.

A term that refers to an OLMC pin marks that OLMC as *fed back*.  If the
OLMC is still *unused*, it becomes *input*.

"Inverted target polarity" means the combined polarity of §5.2.  So for
a pin declared `/R`, `R.E = …` is E19 and `/R.E = …` is accepted.

The `=` is checked after all of the above target checks: if the token
after the target (and suffix) is not `=`, report E14.  So `A B`, with `A`
an input-only pin, reports E15, not E14.

### 6.2 Mode of the GAL16V8 and GAL20V8

These devices have three global modes.  The mode is chosen by the first
rule that applies:

1. If any OLMC is a registered output, the mode is **registered** (mode 3).
2. Otherwise, if any OLMC is a tristate output, the mode is **complex**
   (mode 2).
3. Otherwise, the mode is **complex** if either of these is fed back (an
   *input*, or an undecided output used as a term):
   * on the GAL16V8, the OLMC on pin 15 or 16;
   * on the GAL20V8, the OLMC on pin 18 or 19.

   In simple mode those pins have no path into the array.
4. Otherwise the mode is **simple** (mode 1).

| Mode | SYN | AC0 |
|---|---|---|
| simple | 1 | 0 |
| complex | 1 | 1 |
| registered | 0 | 1 |

Then every undecided output is resolved:
* in simple mode it becomes **combinational**;
* in complex and registered mode it becomes **tristate with a permanently
  enabled output** (§7.3).

### 6.3 GAL22V10 and GAL20RA10

These devices have no global mode.  Undecided outputs become tristate
outputs with permanently enabled output.

### 6.4 Architecture bits

Each OLMC has a *polarity bit* (XOR on the 16V8/20V8, S0 on the
22V10/20RA10):
* 1 if the OLMC is an output (of any kind) and active high;
* 0 otherwise, which includes unused and input OLMCs.

The GAL16V8 and GAL20V8 also have an **AC1** bit per OLMC, and the
GAL22V10 an **S1** bit per OLMC:
* 1 if the OLMC is an *input* or a tristate output;
* 0 otherwise (combinational, registered or unused).

The GAL16V8 and GAL20V8 also have 64 **PT** (product-term disable) bits,
which are always 1.

These per-OLMC bits are stored with the **highest-numbered OLMC pin
first**.  For example, the GAL16V8 XOR fuses 2048 … 2055 belong to pins
19, 18, …, 12.

---

## 7. Fuse map generation

### 7.1 Array organisation

The fuse array has R rows (product terms) of C columns (fuses).  Fuse
number *r*·C + *c* is row *r*, column *c*.

Every input signal reaching the array occupies two adjacent columns:
* the even column *k* is the true signal;
* column *k*+1 is the complement.

A fuse of **0** puts that signal into the row's product term.  A fuse of
**1** leaves it out.  So a row of all 1s is constantly true, and a row of
all 0s is constantly false.

**Initial state.** All array fuses start at 1.

**Literals.** A term adds a literal to the current row by setting one
fuse to 0:
* column *k* for the term's pin, if the term's polarity (§5.2) is not
  inverted;
* column *k*+1 if it is inverted.

### 7.2 Signal columns

The *k* for each pin is given in the tables below.  "–" means the pin has
no column.

**GAL16V8**

| Pin | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| simple | 2 | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 30 | 26 | 22 | 18 | – | – | 14 | 10 | 6 |
| complex | 2 | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 30 | – | 26 | 22 | 18 | 14 | 10 | 6 | – |
| registered | – | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | – | 30 | 26 | 22 | 18 | 14 | 10 | 6 | 2 |

**GAL20V8**

| Pin | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| simple | 2 | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 32 | 36 | 38 | 34 | 30 | 26 | 22 | – | – | 18 | 14 | 10 | 6 |
| complex | 2 | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 32 | 36 | 38 | 34 | – | 30 | 26 | 22 | 18 | 14 | 10 | – | 6 |
| registered | – | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 32 | 36 | – | 38 | 34 | 30 | 26 | 22 | 18 | 14 | 10 | 6 | 2 |

**GAL22V10**

| Pin | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| column | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 32 | 36 | 40 | 42 | 38 | 34 | 30 | 26 | 22 | 18 | 14 | 10 | 6 | 2 |

**GAL20RA10**

| Pin | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| column | – | 0 | 4 | 8 | 12 | 16 | 20 | 24 | 28 | 32 | 36 | – | 38 | 34 | 30 | 26 | 22 | 18 | 14 | 10 | 6 | 2 |

**Pins that cannot be used as terms.**

* *Simple mode.* The 16V8/20V8 pins with no column in simple mode can
  never be reached as terms, because using them forces complex mode
  (§6.2).
* *Complex mode, 16V8 pins 12 and 19.* Using these as a term is E20.
* *Complex mode, 20V8 pins 15 and 22.* Using these as a term is E21.
* *Registered mode, 16V8 pins 1 and 11.* These are the clock and output
  enable; using them as a term is E26.
* *Registered mode, 20V8 pins 1 and 13.* Likewise E27.
* *GAL20RA10 pins 1 and 13.* These are /PL and /OE; using them as a term
  is E37 and E38 respectively.

These checks need the mode, so they happen in the second reading
(§8.2).

**GAL22V10 registered feedback.** When a term refers to an OLMC pin
whose OLMC is a *registered, active-high* output (S1 = 0 and S0 = 1), the
term's polarity is inverted once more before choosing the column.  This
rule applies only to the GAL22V10.

### 7.3 Rows belonging to each OLMC

**GAL16V8 and GAL20V8.**

Rows: each OLMC owns 8 consecutive rows.  The highest OLMC pin owns rows
0–7, the next pin down rows 8–15, and so on:
* GAL16V8: pin 19 → rows 0–7, …, pin 12 → rows 56–63;
* GAL20V8: pin 22 → rows 0–7, …, pin 15 → rows 56–63.

| OLMC kind (after §6.2) | Row 0 of the OLMC | Rows for the sum | Max. product terms |
|---|---|---|---|
| Combinational (simple mode) | part of the sum | 0–7 | 8 |
| Registered (registered mode) | part of the sum | 0–7 | 8 |
| Tristate (complex or registered mode) | output enable | 1–7 | 7 |

**GAL22V10.**

| Rows | Owner | Product terms |
|---|---|---|
| 0 | AR | 1 |
| 1–9 | pin 23 | 8 |
| 10–20 | pin 22 | 10 |
| 21–33 | pin 21 | 12 |
| 34–48 | pin 20 | 14 |
| 49–65 | pin 19 | 16 |
| 66–82 | pin 18 | 16 |
| 83–97 | pin 17 | 14 |
| 98–110 | pin 16 | 12 |
| 111–121 | pin 15 | 10 |
| 122–130 | pin 14 | 8 |
| 131 | SP | 1 |

The first row of each OLMC is its output enable.  The remaining rows hold
the sum, so the product-term limit is the number of rows minus one.

**GAL20RA10.** Each OLMC owns 8 rows: pin 23 → rows 0–7, pin 22 → 8–15,
…, pin 14 → 72–79.  Within an OLMC whose first row is *b*:

| Row | Use |
|---|---|
| *b* | output enable (`.E`) |
| *b*+1 | clock (`.CLK`) |
| *b*+2 | asynchronous reset (`.ARST`) |
| *b*+3 | asynchronous preset (`.APRST`) |
| *b*+4 … *b*+7 | sum: at most 4 product terms |

### 7.4 Writing an equation into the array

Equations are read a second time, in file order, now with the mode and
every OLMC's final kind known.

**Control equations** are `.E`, `.CLK`, `.ARST`, `.APRST`, and AR and SP on
the GAL22V10.  Each writes exactly one row: the dedicated row from §7.3.
All of its terms are ANDed into that row.  An OR in a control equation is
E29, reported at the term that follows the OR.

**Sum equations** are all other equations.  They write their product
terms into consecutive rows starting at the OLMC's first sum row (§7.3):
* each OR moves to the next row;
* going past the last row of the OLMC is E30, reported at the term after
  the OR that overflowed.

After the last term, every remaining sum row of the OLMC is set to all
0s.  Those rows are then constantly false and do not affect the output.

**Checks on each term.** These are done in this order:

1. The mode or device pin restrictions of §7.2 (E20, E21, E26, E27, E37,
   E38).
2. If the term is the VCC or GND pin:
   * negated → E25;
   * any operator before it in the same equation, or directly after it →
     E28;
   * `GND`: the current row is set to all 0s;
   * `VCC`: the current row is left as it is (all 1s).
3. Otherwise the literal is written as in §7.1.

**Rows that are not written stay all 1s.** As a result:
* a tristate output with no `.E` equation is always enabled, because its
  enable row stays true;
* on the GAL20RA10, a combinational output has its reset and preset rows
  left true.  That is how this device selects combinational
  (register-bypass) operation.

### 7.5 Final clean-up

After all equations have been written:

1. **Unused and input OLMCs.** Every row of each OLMC that is *unused* or
   *input* is set to all 0s.
2. **GAL22V10 AR and SP.** If AR is not defined, row 0 is set to all 0s.
   If SP is not defined, row 131 is set to all 0s.
3. **GAL20RA10.** Process each OLMC that is not *unused* (input OLMCs
   included), from pin 14 upwards:
   1. If it is a registered output with no `.CLK`, report error E41 for
      that pin (§8.3) and stop.
   2. If it has no `.CLK`, set row *b*+1 to all 0s.
   3. If it is a registered output, set row *b*+2 to all 0s when there is
      no `.ARST`, and row *b*+3 to all 0s when there is no `.APRST`.

---

## 8. Errors

### 8.1 Error list

Assembly stops at the first error.  The error identifiers below are used
by this document and are also the error numbers of the reference
implementation.  The wording of messages is up to the implementation.

| Id | Condition |
|---|---|
| E1 | Line 1 does not start with a supported device type followed by space, TAB or LF |
| E2 | Unexpected end of file |
| E3 | Negation sign in a pin declaration not followed by a name |
| E4 | Pin name longer than 8 characters |
| E5 | Illegal character in the pin declarations |
| E6 | `GND` or `VCC` declared on the wrong pin |
| E7 | VCC pin not declared as `VCC` |
| E8 | GND pin not declared as `GND` |
| E9 | Pin name declared twice |
| E10 | Negation sign inside a pin name |
| E11 | Unknown pin name in an equation |
| E12 | `NC` used in an equation |
| E13 | Unknown suffix |
| E14 | `=` expected |
| E15 | Equation target has no OLMC |
| E16 | Output defined more than once |
| E17 | `.E` before the output is defined |
| E18 | GAL22V10: `AR` or `SP` declared as a pin name |
| E19 | `.E`, `.CLK`, `.ARST` or `.APRST` with inverted target polarity (§6.1) |
| E20 | GAL16V8 complex mode: pin 12 or 19 used as a term |
| E21 | GAL20V8 complex mode: pin 15 or 22 used as a term |
| E22 | `.E` defined twice for one output |
| E23 | GAL16V8/20V8: `.E` for a registered output |
| E24 | `.E` for an output defined without `.T` |
| E25 | Negated `VCC` or `GND` |
| E26 | GAL16V8 registered mode: pin 1 or 11 used as a term |
| E27 | GAL20V8 registered mode: pin 1 or 13 used as a term |
| E28 | `VCC` or `GND` combined with other terms |
| E29 | More than one product term in a control equation |
| E30 | Too many product terms for the OLMC |
| E31 | GAL22V10: `AR` or `SP` used as a term |
| E32 | GAL22V10: negated `AR` or `SP` |
| E33 | No equations |
| E34 | `.CLK` on a device other than the GAL20RA10 |
| E35 | `.ARST` on a device other than the GAL20RA10 |
| E36 | `.APRST` on a device other than the GAL20RA10 |
| E37 | GAL20RA10: pin 1 used as a term |
| E38 | GAL20RA10: pin 13 used as a term |
| E39 | GAL22V10: suffix on `AR` or `SP` |
| E40 | GAL22V10: `AR` or `SP` defined twice |
| E41 | GAL20RA10: registered output without `.CLK` |
| E42 | `.CLK` before the output is defined |
| E43 | `.ARST` before the output is defined |
| E44 | `.APRST` before the output is defined |
| E45 | `.CLK` defined twice for one output |
| E46 | `.ARST` defined twice for one output |
| E47 | `.APRST` defined twice for one output |
| E48 | `.CLK`, `.ARST` or `.APRST` for an output that is not registered |

### 8.2 Which error is reported first

The checks happen in three phases.  The first error in the earliest
phase wins.

**Phase A: read and classify.** This covers:
* lines 1 and 2;
* the pin declarations (§4);
* the syntax and classification of every equation in file order: E11–E19,
  E22–E24, E31–E36, E39, E40, E42–E48, plus E2 and E33.

**Phase B: write the array.** Every equation in file order, with the
term checks of §7.4: E20, E21, E25–E30, E37, E38.

**Phase C: clean-up.** E41.

So a phase-A error late in the file is reported in preference to a
phase-B error earlier in the file.  A phase-B error is reported even when
the equation that set the mode comes later.

### 8.3 Error report format and line numbers

**Format.** Errors are reported on the console in one of two forms:

```
Error in line N: <message>
Error, pin P: <message>
```

The second form is used only for E41, where P is the pin of the
offending output.  The reference writes to standard output; an
implementation MAY use standard error.  The test suite looks only for
these two prefixes.

**Line N** is assigned as follows:

| Kind of error | N is the line of… |
|---|---|
| E1 | line 1 |
| Pin declaration errors | the line where the offending pin name is |
| Errors about the target name itself (E11, E12, E32) | the target name |
| Other errors about an equation's target: suffix and classification errors (E13–E17, E19, E22–E24, E34–E36, E39, E40, E42–E48) | the first token after the target name (normally the `.` or `=`, usually on the same line) |
| Errors about a term (E11, E12, E20, E21, E25–E31, E37, E38) | the term |
| E33 | `DESCRIPTION` |
| E2 | unspecified; tests only require failure |

---

## 9. Output file formats

All four files are text files written with the platform's native line
endings, except that `-w` forces CR LF in the JEDEC file.  In the
descriptions below `\n` stands for one line ending.

### 9.1 Fuse numbering

Each device's fuses are numbered as follows.

**GAL16V8** (2194 fuses)

| Fuses | Contents |
|---|---|
| 0–2047 | array |
| 2048–2055 | XOR |
| 2056–2119 | signature |
| 2120–2127 | AC1 |
| 2128–2191 | PT |
| 2192 | SYN |
| 2193 | AC0 |

**GAL20V8** (2706 fuses)

| Fuses | Contents |
|---|---|
| 0–2559 | array |
| 2560–2567 | XOR |
| 2568–2631 | signature |
| 2632–2639 | AC1 |
| 2640–2703 | PT |
| 2704 | SYN |
| 2705 | AC0 |

**GAL22V10** (5892 fuses)

| Fuses | Contents |
|---|---|
| 0–5807 | array |
| 5808–5827 | S0/S1 pairs: S0 then S1 of pin 23, then pin 22, …, pin 14 |
| 5828–5891 | signature |

**GAL20RA10** (3274 fuses)

| Fuses | Contents |
|---|---|
| 0–3199 | array |
| 3200–3209 | S0 of pins 23 … 14 |
| 3210–3273 | signature |

### 9.2 JEDEC file (`.jed`)

The file consists of these lines, in order.  Spaces shown are literal.

```
<STX>\n                              (omitted with -a)
Used Program:   GALasm 2.1\n
GAL-Assembler:  GALasm 2.1\n
Device:         <device>\n            <device> = GAL16V8 | GAL20V8 | GAL22V10 | GAL20RA10
\n
*F0\n
*G0\n                                 (*G1 with -s)
*QF<total fuses>\n                    2194 | 2706 | 5892 | 3274
<array lines>
<architecture lines>
*C<fuse checksum>\n
*\n
<ETX><transmission checksum>\n       (omitted with -a)
```

The header lines in detail:
* **Program lines.** `Used Program:` is followed by 3 spaces,
  `GAL-Assembler:` by 2 spaces, and `Device:` by 9 spaces.  The values of
  the first two lines identify the program.  An implementation MAY write
  its own name and version there (the test suite ignores them), but the
  labels and spacing MUST be as shown.
* **`*F0`** declares that fuses not listed are 0.
* **Fuse lines.** Each starts with `*L`, then the address of its first
  fuse as a decimal number of at least 4 digits (zero-padded, `%04d`),
  then a space, then the fuse values as the characters `0` and `1`.

**Array lines.** One line per array row that contains at least one 1, in
row order.  Each line holds the C fuses of that row and is labelled with
the row's first fuse number.  Rows of all 0s are omitted.

**Architecture lines.** These are always present, even when all their
fuses are 0.

| Device | Lines, in order |
|---|---|
| GAL16V8, GAL20V8 | XOR (8 fuses), signature (64), AC1 (8), PT (64), SYN (1), AC0 (1): six lines |
| GAL22V10 | the 20 S0/S1 fuses, then the signature (64): two lines |
| GAL20RA10 | the 10 S0 fuses, then the signature (64): two lines |

### 9.3 Example

For a GAL16V8 with signature `HAND`, pins `A B C D E F G H I GND J K L M N O
P Q R VCC`, and the single equation `R = A * /B`, the file is:

```
<STX>
Used Program:   GALasm 2.1
GAL-Assembler:  GALasm 2.1
Device:         GAL16V8

*F0
*G0
*QF2194
*L0000 10011111111111111111111111111111
*L2048 10000000
*L2056 0100100001000001010011100100010000000000000000000000000000000000
*L2120 00000000
*L2128 1111111111111111111111111111111111111111111111111111111111111111
*L2192 1
*L2193 0
*C0d18
*
<ETX>45be
```

How the example's lines come about:
* **Array.** Pin 19 owns rows 0–7.  In simple mode `A` (pin 1) is column 2
  and `B` (pin 2) is column 0.  So row 0 has fuse 2 (A true) and fuse 1
  (B complement) at 0.  Rows 1–63 are all 0 and omitted.
* **XOR.** The output on pin 19 is active high, so the first XOR fuse is
  1.

`45be` is the transmission checksum (§9.5), computed for the file as
shown, with LF line endings.  This example is the test case
`16v8_hand_derived`.

### 9.4 Fuse checksum (`*C`)

1. Take all fuses of the device in fuse-number order, from 0 to
   total − 1.
2. Pack them into bytes, 8 at a time: fuse 8*i* is bit 0 (least
   significant) of byte *i*, fuse 8*i*+7 is bit 7.  If the total is not a
   multiple of 8, the last byte is padded with zeros.
3. Add the bytes, modulo 65536.
4. Write the result as exactly 4 lowercase hexadecimal digits.

The checksum covers every fuse, including the architecture fuses and the
signature.  It is written whether or not `-a` is given.

### 9.5 Transmission checksum

Without `-a`:
* The file begins with `<STX>` and a line ending.
* After the closing `*` line comes `<ETX>`, immediately followed by the
  transmission checksum and a line ending.

The transmission checksum is the sum, modulo 65536, of the bytes of the
file as written, from `<STX>` to `<ETX>` inclusive.  It includes any CR
bytes of CR LF line endings.  It is written as exactly 4 lowercase
hexadecimal digits.

### 9.6 Effect of `-a`

With `-a`:
* `<STX>` and its line ending are omitted, so the file starts with
  `Used Program:`;
* the file ends with the `*` line: no `<ETX>` and no transmission
  checksum.

### 9.7 Fuse listing (`.fus`)

The listing contains, in order:

1. **GAL22V10 only, first:** two line endings, the text `AR`, then row 0
   in the row format below.
2. **For each OLMC, from the highest pin down** (16V8: 19…12; 20V8:
   22…15; 22V10 and 20RA10: 23…14):
   * two line endings, then `Pin %2d = ` followed by the declared pin
     name (including any negation sign);
   * spaces to pad the name to 13 characters (none if it is longer);
   * then, depending on the device:
     * 16V8/20V8: `XOR = <x>   AC1 = <a>`;
     * 22V10: `S0 = <x>   S1 = <s>`;
     * 20RA10: `S0 = <x>`.

     There are three spaces before `AC1` and `S1`; `<x>`, `<a>`, `<s>`
     are single digits;
   * then each row owned by the OLMC (8 rows, or the OLMC's row count on
     the 22V10, enable row included) in the row format.
3. **GAL22V10 only, after the OLMC on pin 14:** two line endings, the
   text `SP`, then row 131 in the row format.
4. **Finally**, two line endings.

**Row format.** A line ending, then the row number right-aligned in 3
characters, then a space.  Then for each column *c* from 0:
* a space if *c* is a multiple of 4;
* then `-` if the fuse is 1, or `x` if it is 0.

So `x` marks a connected literal.

Example (GAL22V10, AR = H * /I):

```


AR
  0  ---- ---- ---- ---- ---- ---- ---- ---- x--- -x-- ----

Pin 23 = Q9           S0 = 0   S1 = 0
  1  xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx
```

### 9.8 Pin listing (`.pin`)

```
\n
\n
 Pin # | Name     | Pin Type\n
-----------------------------\n
```

That is 29 dashes.  Then one line per pin, from 1 to N:

```
  %2d   | <name><pad>| <type>\n
```

* `<name>` is the declared name, including any negation sign.
* `<pad>` is spaces bringing the name to 9 characters (none if it is
  longer).
* After the VCC pin's line, one extra line ending follows.

`<type>` is decided by the first rule that matches:

| Pin | Type |
|---|---|
| the GND pin | `GND` |
| the VCC pin | `VCC` |
| 16V8/20V8 in registered mode, pin 1 | `Clock` |
| 16V8 in registered mode, pin 11; 20V8 in registered mode, pin 13 | `/OE` |
| GAL22V10 pin 1 | `Clock/Input` |
| an OLMC pin whose OLMC is *input* | `Input` |
| an OLMC pin whose OLMC is any kind of output | `Output` |
| an OLMC pin whose OLMC is *unused* | `NC` |
| any other pin (including 20RA10 pins 1 and 13) | `Input` |

### 9.9 Chip diagram (`.chp`)

The file contains, in order:

1. Two line endings.
2. 31 spaces, then the device title, then two line endings.  The title is
   ` GAL16V8`, ` GAL20V8`, ` GAL22V10` (each with one leading space) or
   `GAL20RA10` (no leading space).
3. 26 spaces, then `-------\___/-------`, then a line ending.
4. For *n* = 1 … N/2, with L = name of pin *n* and R = name of pin
   N+1−*n*:
   * (25 − length of L) spaces (none if negative), then L, then
     ` | `, then *n* as `%2d`, then 11 spaces, then N+1−*n* as `%2d`,
     then ` | `, then R, then a line ending;
   * if *n* < N/2: 26 spaces, then `|                 |` (17 spaces
     between the bars), then a line ending.
5. 26 spaces, then 19 dashes, then a line ending.

---

## 10. Console output *(non-normative except §8.3)*

The reference prints, in order:
* a banner;
* a line per assembly pass;
* a summary such as `GAL16V8; Operation mode: simple; Security fuse off`
  (the mode is shown for the 16V8/20V8 only);
* finally, either a success or a failure line.

With `-v` it also explains the mode choice for the 16V8/20V8, naming each
pin that forced it.  Implementations are free to word all of this as
they like.  The only requirement is that errors use the prefixes of
§8.3.

---

## 11. Behaviour this document leaves open

The reference shows the following behaviour.  It is not specified here
and not tested; implementations MAY differ:

* The exact usage, help and error message texts.
* Exit status values other than zero versus non-zero.
* The message for a missing input file.  The reference reports "not
  enough memory", which is misleading.
* Behaviour on an empty input file.  The reference reads past the end of
  its buffer, so behaviour is undefined.
* The line number reported for unexpected end of file.
* Handling of source bytes ≥ 0x80 inside names.

---

## 12. Recommended improvements *(non-normative)*

These changes would not break any test in the suite.  They would make an
implementation more robust than the reference:

1. **Accept CR before LF anywhere.** That includes the byte after the
   device type and the end of the signature, so files with Windows line
   endings work on every platform.
2. **Never read past the end of the input.** At end of file, report E2
   cleanly.
3. **Clear message for a missing input file.** Report that the input file
   cannot be opened, and exit with a non-zero status.
4. **Directory names.** Only look for the extension in the final path
   component when naming output files.
5. **Don't leave partial output.** If an output file cannot be written,
   report it and exit non-zero without leaving partial files behind.

---

## 13. Glossary

* **OLMC.** Output Logic Macrocell: the configurable output stage behind
  an output pin.
* **Product term.** One row of the AND array: the AND of the literals
  whose fuses are 0.
* **Sum.** The OR of an OLMC's sum rows, which drives the output.
* **Enable row.** The product term controlling an OLMC's tristate buffer.
* **JEDEC file.** The industry-standard fuse-map file format (JESD3)
  understood by device programmers.
