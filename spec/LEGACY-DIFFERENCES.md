# Differences from GALasm 2.1

`GALASM-SPEC.md` describes how a GALasm-compatible assembler should
behave.  In a few places it deliberately departs from GALasm 2.1, the
existing code, which this file calls the *legacy* assembler.  This file
records each departure, what the legacy assembler does, and why the
change is safe.

Each change meets one of two tests:

* it accepts input the legacy assembler rejected, or improves an error
  message, without changing the output for any file the legacy assembler
  already handled; or
* it only affects behaviour that does not materially matter to users.

Everything in this directory is under the MIT license (see `LICENSE`).

## Testing the legacy assembler

Test cases for the improved behaviour carry a `legacy_skip` note in their
`case.json`.  Run the suite with `--legacy` to skip them when testing the
legacy code; `make check` in `src/` does this.  Without `--legacy`,
every case applies.

## Changes

### 1. Source files with Windows (CR LF) line endings

* **Legacy.** The byte after the device type on line 1 must be a space,
  TAB or LF.  On Linux and macOS a CR LF file therefore fails with "type
  of GAL expected" on line 1.  On Windows the C runtime strips the CR
  before the assembler sees it, so the same file works there.
* **Spec (§3.1–§3.3).** CR LF is accepted everywhere and gives exactly the
  same result as LF.  The CR is never part of the signature.
* **Why it is safe.** This only accepts files that were rejected before,
  and it matches what Windows users already get.
* **Tests.** `16v8_crlf_source` (skipped for legacy), checked against its
  LF twin `16v8_crlf_source_lf_twin`.

### 2. Missing input file

* **Legacy.** Prints "Error: Not enough free memory!".
* **Spec (§2.3).** The message says the file cannot be opened and names
  it.
* **Why it is safe.** It changes only the message text.
* **Tests.** `cli_missing_file` (skipped for legacy).

### 3. Negation sign on the target of `.E`, `.CLK`, `.ARST`, `.APRST`

* **Legacy.** Error 19 ("not allowed to be negated") whenever the
  target's combined polarity is inverted.  That combines the `/` in the
  equation with the `/` in the pin declaration.  So:
  * for a pin declared `R`, `R.E` is required and `/R.E` is an error;
  * for a pin declared `/R`, `/R.E` is required and `R.E` is an error.

  The negation never affected the fuses: it was only checked.
* **Spec (§6.1).** The negation sign is allowed and ignored in both
  cases.  Error 19 is retired.
* **Why it is safe.** Every file the legacy assembler accepted produces
  the same output.  Files it rejected for this reason now work.
* **Tests.**
  * `16v8_enable_target_negated` (skipped for legacy);
  * `16v8_enable_declared_negated_plain` (skipped for legacy);
  * `20ra10_clk_target_negated` (skipped for legacy);
  * `16v8_enable_declared_negated`, which applies to both.

  The removed test cases `err_negated_enable`, `err_enable_declared_negated`
  and `err_20ra10_negated_clk` used to check for error 19.

### 4. Output file names when a directory name contains a dot

* **Legacy.** The extension is found by searching the whole path for the
  last `.`.  So for input `dir.v2/design` the outputs are written as
  `dir.jed` etc., in the parent directory.
* **Spec (§2.2).** Only the file name is examined, so the outputs are
  `dir.v2/design.jed` etc.
* **Why it is safe.** It only matters for an input whose file name has no
  extension and whose path includes a dotted directory.  The old result
  is clearly a bug.
* **Tests.** `cli_directory_with_dot` (skipped for legacy), and
  `cli_directory_with_dot_and_ext`, which applies to both.

### 5. Reading past the end of the input

* **Legacy.** When the input ends in the middle of the pin list or the
  equations (for example a missing `DESCRIPTION`), the scanner reads one
  byte beyond its buffer before noticing.  The outcome depends on that
  stray byte, but in practice it is always a failure.
* **Spec (§5.1).** It MUST NOT read beyond the input, and reports
  "unexpected end of file".
* **Why it is safe.** These inputs fail either way.
* **Tests.** The `err_eof_*` and `err_no_description` cases check only
  that assembly fails.

### 6. Which error is reported when a file has several

* **Legacy.** Errors are found in three phases, and the first error of
  the earliest phase is reported:
  1. reading and classifying all equations;
  2. checks that depend on the GAL16V8/20V8 mode;
  3. the GAL20RA10 missing-clock check.

  So a syntax error late in the file wins over a mode error early in the
  file.
* **Spec (§8.2).** Any one of the errors may be reported.
* **Why it is safe.** The file is rejected either way, and the user fixes
  errors one at a time.
* **Tests.** `err_classification_errors_first` accepts either error's
  line.

### 7. Line number for errors about an output definition

* **Legacy.** Errors such as "output defined twice" or "unknown suffix"
  are reported at the line of the token after the output name (the `.`
  or `=`).
* **Spec (§8.3).** Either the line of the output name or the line of that
  next token.  The two differ only when an equation is split right after
  its output name.
* **Tests.** `err_output_error_line_of_equals` accepts either line.

### 8. Options without a leading `-`

* **Legacy.** In some positions an argument without a `-` is still parsed
  as options: `galasm sa file.pld` acts like `galasm -a file.pld`, with
  the first letter silently ignored.
* **Spec (§2.1).** Options always start with `-`.
* **Why it is safe.** This looks accidental and is undocumented.  A
  command relying on it would now get a usage error rather than wrong
  output.
* **Tests.** None.

## Legacy behaviour that is kept

These oddities stay in the specification, because removing them could
change or reject files that work today:

* **Suffixes:** `.T`, `.R` and `.E` are recognised by their first letter,
  so `.Tri` and `.Enable` work.
* **End of equations:** `DESCRIPTION` is recognised as a prefix, so
  `DESCRIPTIONS` also ends the equations.
* **Signature:** the signature stops at a TAB but includes spaces.
* **22V10 reserved names:** on the GAL22V10 only the exact names `AR`
  and `SP` are rejected as pin names; `/AR` is accepted.
* **Output formatting:** the `.chp` title for the GAL20RA10 has no
  leading space, the `.pin` file ends with an extra blank line, and the
  `.jed` header names the program "GALasm 2.1".  Users may diff these
  files.
* **GAL22V10 registered feedback:** the polarity inversion for
  active-high registered feedback (spec §7.2) stays.  It is needed for
  the correct fuse map; it is not a quirk.

## Bringing the legacy code in line

All four testable changes (1–4) are small edits to the legacy C code.
They were tried out in a scratch copy, which passed the whole suite
without `--legacy`.  Porting them would let the legacy assembler drop the
`legacy_skip` markers.  This has not been done, because the legacy code
is kept as it was.
