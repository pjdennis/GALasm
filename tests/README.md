# GALasm conformance tests

A black-box test suite for GALasm-compatible GAL assemblers.  It runs the
assembler as a command-line program and checks its exit status, its error
reports and the `.jed`, `.fus`, `.pin` and `.chp` files it writes.  It knows
nothing about how the assembler is implemented, so the same suite checks the
original GALasm 2.1 code and any reimplementation.

Everything in this directory is under the MIT license (see `LICENSE`).

## Running

Requires Python 3 (standard library only).

```sh
cd src && make && make check                 # build, then run the suite
python3 tests/run_tests.py                   # test src/galasm
python3 tests/run_tests.py --galasm path/to/other-assembler
python3 tests/run_tests.py -k 22v10 -v       # only cases whose name contains "22v10"
python3 tests/run_tests.py --legacy          # skip cases where the spec improves on GALasm 2.1
```

The suite tests the behaviour in `spec/GALASM-SPEC.md`.  A few cases
cover deliberate improvements over GALasm 2.1, listed in
`spec/LEGACY-DIFFERENCES.md`.  Those cases carry a `legacy_skip` note,
and `--legacy` skips them; `make check` in `src/` uses it for the
GALasm 2.1 code.

The runner exits with status 0 when every case passes.

## Case format

Each directory in `cases/` is one case:

| File | Purpose |
|---|---|
| `case.json` | What to run and what to expect |
| `input.pld` | Source file, copied into an empty temporary directory before the run |
| `expected.jed` / `.fus` / `.pin` / `.chp` | Expected output files (successful cases only) |

`case.json` keys:

| Key | Meaning |
|---|---|
| `description` | One line saying what the case covers |
| `expect` | `success`: exit status 0 and the expected files; `error`: non-zero exit status and no output files; `usage`: non-zero exit status and no output files; `help`: exit status 0 and no output files |
| `args` | Command-line arguments; `{input}` is replaced by the input file name. Default `["{input}"]` |
| `input` | Name to give the copied input file; may include a directory. Default `input.pld` |
| `outputs` | Output extensions that must be produced, and no others. Default: all four for `success`, none otherwise |
| `error_line` | The console output must contain `Error in line N:` with this N, or with one of the Ns if a list is given |
| `error_pin` | The console output must contain `Error, pin N:` with this N |
| `crlf` | The `.jed` file must use CR LF line endings throughout |
| `must_mention` | Strings the console output must contain |
| `legacy_skip` | Why GALasm 2.1 fails this case; `--legacy` skips it |

## How outputs are compared

* **Line endings.** CR LF is converted to LF before comparing, so native Windows line endings are accepted. For cases with `"crlf": true`, the `.jed` file must use CR LF on every line.
* **JEDEC transmission checksum.** The four hex digits after `<ETX>` are checked against the byte sum of the file as written, from `<STX>` to `<ETX>` inclusive. They are then left out of the comparison.
* **Program name.** The values of the `Used Program:` and `GAL-Assembler:` header lines are ignored, so another implementation may put its own name and version there.
* **Everything else.** The rest of every file must match byte for byte, including the fuse checksum (`*C`).

The console output is only checked for the `Error in line N:` and
`Error, pin N:` prefixes. Banner, progress and error-message text are free.

## Where the expectations come from

* **Inputs.** Every `input.pld` was written from scratch for this suite. None is taken from the GALer or GALasm distributions.
* **Expected outputs.** These were captured with `run_tests.py --update` from GALasm 2.1 as maintained at <https://github.com/pjdennis/GALasm> (commit `1b5ef79`). There are two exceptions:
  * `16v8_hand_derived`, whose expected files were worked out by hand from the device architecture and the JEDEC format. That case cross-checks the captured outputs.
  * The `legacy_skip` success cases, whose expected files come from GALasm 2.1 assembling an equivalent input it accepts. For example, the CR LF source uses its LF twin's output.
* **Error line numbers.** These were predicted by hand first and then confirmed against the reference.

`--update` only writes expected files that are missing. To regenerate one
on purpose, delete it and run `--update` with the reference assembler.

## Adding a case

1. Create `cases/<name>/input.pld` and `cases/<name>/case.json`.
2. For a success case, run `python3 tests/run_tests.py --update -k <name>`
   with the reference assembler.
3. Read the generated files and convince yourself they are right before
   committing them.

Name cases by device or area (`16v8_`, `20v8_`, `22v10_`, `20ra10_`,
`cli_`, `err_`).
