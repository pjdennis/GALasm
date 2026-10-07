# GALasm specification and clean-room plan

This directory holds a behavioural specification of the GALasm assembler,
written so that a new, MIT-licensed implementation can be built without
reference to the existing source code.  The conformance suite in `../tests/`
is its executable counterpart.  `LEGACY-DIFFERENCES.md` lists the few
places where the specification deliberately improves on GALasm 2.1.

Everything in this directory is under the MIT license (see `LICENSE`).

## Why

The existing GALasm source has no open-source license:

* GALasm 2.1 is "Copyright (c) 1998-2003 Alessandro Zummo. All Rights
  Reserved. Commercial use is strictly forbidden".
* Its assembler core is a port of Christian Habermann's GALer (1991–96).
  GALer was shareware and allowed redistribution only unaltered.
* Zummo's README says the port was made without Habermann's permission.
  See also daveho/GALasm issue #5.

A clean-room reimplementation gives the project an assembler it can
license permissively.  The current code is kept alongside it for people
who prefer it.

## Process

There are two roles, and they must be kept apart.

**Specification side.** This side may read the old code.  It writes and
maintains `spec/` and `tests/`, which describe behaviour only: inputs,
outputs, formats and rules.  It never includes code, code structure,
identifiers, comments or message wording from the old sources.

**Implementation side.** This side MUST NOT see the old code.  It works
from `spec/` and `tests/` only:

* in a repository or checkout that contains nothing else;
* without looking up the GALasm or GALer sources, or other GAL
  assemblers' sources, online;
* without copying the GALer documentation (`galer/`) or the example
  `.pld` files (`examples/`), which are Habermann's work.

**Questions.** When the implementer finds the spec unclear, they ask.
The specification side answers by improving the spec, and by adding a
test case where possible, never by describing the old code.  Keep these
exchanges, for example as issues or pull requests against `spec/`, as a
record of the process.

**Done means:**
1. `python3 tests/run_tests.py --galasm <new binary>` passes on Linux,
   macOS and Windows.
2. A similarity check between the new and old sources finds no copied
   structure beyond what the spec itself dictates.  Use a tool such as
   JPlag or MOSS, plus a manual review.  Keep the report.

### A caveat about AI-written code

Language models may have seen GALasm's public source during training.
That weakens the "never saw the code" argument for an AI implementer
compared with a human one.  These mitigations help:
* write behaviour-only specs;
* give the implementer instructions not to recall or reproduce existing
  GALasm/GALer code;
* run the similarity check above.

Whether that is enough is a legal judgement, not a technical one.

## Suggested hand-off to an implementing agent

Start the implementer in a fresh repository that contains only copies of
`spec/` and `tests/`, then give it a prompt along these lines:

> Implement a GAL assembler in portable C (C99, no dependencies beyond
> the standard library) that conforms to `spec/GALASM-SPEC.md`.  The
> conformance suite is `tests/run_tests.py`; build a `galasm` executable
> and make every case pass.  Work test-first: pick a failing case, make it
> pass, refactor, repeat.  Add unit tests for internal pieces as you go.
> This is a clean-room implementation: do not look at, search for, or
> recall the source code of GALasm, GALer or any other GAL assembler, and
> do not use their documentation; rely only on this specification, the
> tests and public device datasheets.  If the spec is ambiguous, stop and
> ask rather than guessing.  License the result under MIT.

## Repository layout once the new implementation lands *(proposal)*

* `src/`: the new MIT implementation, the default build.
* `legacy/`: the existing GALasm 2.1 code, `galer/` docs and `examples/`,
  unchanged and still under their original terms, with a notice saying
  so.
* `spec/` and `tests/`: shared by both, so the two stay
  behaviour-compatible.  CI runs the suite against both builds.
* A top-level `LICENSE` (MIT) that states it does not cover `legacy/`.
