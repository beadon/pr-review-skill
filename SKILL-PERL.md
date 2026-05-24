---
name: pr-review-perl
description: Perl-specific review checks to augment /pr-review for Perl codebases (.pl, .pm, .t, .in files)
version: "1.0.0"
---

## Overview

Apply these Perl-specific checks in addition to the general `/pr-review` checklist when the diff contains `.pl`, `.pm`, `.t`, or `.in` files. Each check maps to a category in the main skill's table.

---

## Namespace and Symbol Resolution

**Correctness**

- Unqualified built-in calls (`sleep`, `rand`, `print`, `open`, etc.) resolve to `CORE::`, not to any package. A test stub that redefines `*Package::sleep` will never intercept production code that calls `sleep` without qualification. Correct interception requires `*CORE::GLOBAL::sleep` or calling the function with full package qualification.
- `Package::func()` (function call) and `$obj->func()` (method call) dispatch differently. Confirm the right form is used for the context — especially for functions inherited or overridden in subclasses.

---

## Type Coercion and Numeric Behavior

**Correctness**

- `sleep()`, array indices, `sprintf "%d"`, and bitwise operators silently truncate floats. `sleep(0.7)` becomes `sleep(0)`, which returns 0 immediately — a common source of infinite loops when the remaining time is fractional. Use `int()` explicitly when truncation is intended.
- `rand($n)` returns a float in `[0, $n)`. Adding it to an integer produces a float. If downstream code assumes integer arithmetic (e.g., passing the result to `sleep`), wrap in `int(rand($n))`.
- `//` (defined-or) vs `||` (false-or): use `//` when checking for definedness. `||` treats `0` and `""` as false, silently swallowing valid zero or empty-string values.

---

## Strictness and Safety

**Correctness / Security**

- `use strict; use warnings;` must appear in every new file. Absence is a blocker.
- Prefer three-argument `open`: `open(my $fh, '<', $file)`. Two-argument `open(FH, $file)` allows shell injection when `$file` starts or ends with `|`. Flag any two-argument opens on paths influenced by external input.
- `local` vs `my`: `local` gives dynamic (caller-visible) scope to a global variable; `my` gives lexical scope. Confusing them causes action-at-a-distance bugs that are hard to trace.
- User input interpolated directly into a regex (`/$user_input/`) is a code injection risk. Unanchored patterns and patterns with catastrophic backtracking potential should also be flagged.

---

## Error Handling

**Correctness**

- `$@` is reset by any subsequent call, including `eval` itself. Always capture it immediately: `my $err = $@; if ($err) { ... }`.
- Is `die` vs `warn` severity appropriate? Recoverable conditions should warn; unrecoverable ones should die.
- `chomp` is commonly forgotten on values read from files, pipes, or `qx//`. Check that newlines are stripped before values are used in comparisons or concatenation.

---

## References and Data Structures

**Correctness**

- Autovivification: accessing `$h{a}{b}` when `$h{a}` does not exist silently creates an empty hashref at `$h{a}`. In conditional branches this can corrupt data structures unexpectedly.
- Dereferencing an undef reference (`@{$ref}` when `$ref` is undef) throws "Not an ARRAY reference" at runtime. Check definedness before dereferencing, especially for values from hashes or function returns.

---

## Testing Patterns

**Testing**

- `done_testing()` vs `plan`: a missing `done_testing()` at the end means a silent apparent pass if the file exits early. Prefer `done_testing()` unless the count is known and fixed.
- `BAIL_OUT` is correct for fatal preconditions (module fails to load). It is not appropriate for individual test failures — use `die` or `SKIP` instead.
- Stubs via typeglob assignment (`*Package::func = sub { ... }`) should be wrapped in `{ no warnings 'redefine'; ... }`. Prefer scoped restoration (`local *Package::func = sub { ... }`) so the stub does not leak into subsequent tests.
- A stub that never actually gets called provides no signal. After the code under test runs, assert that the stub was called the expected number of times with the expected arguments.

---

## References

- **ddclient/ddclient PR #888 review** — namespace mismatch (`CORE::sleep` vs `ddclient::sleep`) and silent float truncation (`sleep(rand($n))`) discovered during real-world review
- **perldoc perlref** — autovivification and reference safety
- **perldoc perltrap** — common Perl traps including `//` vs `||`, `local` vs `my`
- **perldoc -f open** — three-argument open and injection risk
