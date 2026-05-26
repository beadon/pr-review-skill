---
name: pr-review-perl
description: Perl-specific review checks to augment /pr-review for Perl codebases (.pl, .pm, .t, .in files)
version: "1.1.0"
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

## Syntax and Deprecations

**Style / Correctness**

- `%$hash{key}` is a one-element hash slice, deprecated since Perl 5.20, and generates warnings under `use warnings`. Replace with `$hash->{key}` (arrow dereference). Equally, `@$array[$i]` should be `$array->[$i]`. Flag any `%$` or `@$` derefs followed by a brace subscript — they are never intentional in modern Perl.
- `<<EoTEXT` (bare label) is a non-interpolating heredoc — equivalent to single quotes. Variables like `${program}` and `$var` appear literally in the output. `<<"EoTEXT"` enables interpolation. Flag mismatches between the author's apparent intent (using `$var` inside the heredoc body) and the bare quoting form: the variable expands to nothing, silently producing wrong output.

---

## Loop Control in Iterator Loops

**Correctness**

- In loops that process a list of independent items (hosts, records, queue entries), `last` exits the entire loop and silently abandons all remaining items. Unless that is the explicit intent, use `next` to skip the current item and continue processing the rest. This is especially critical in protocol update loops where a single failure should not prevent other entries from being handled.

---

## Testing Patterns

**Testing**

- `done_testing()` vs `plan`: a missing `done_testing()` at the end means a silent apparent pass if the file exits early. Prefer `done_testing()` unless the count is known and fixed.
- `BAIL_OUT` is correct for fatal preconditions (module fails to load). It is not appropriate for individual test failures — use `die` or `SKIP` instead.
- Stubs via typeglob assignment (`*Package::func = sub { ... }`) should be wrapped in `{ no warnings 'redefine'; ... }`. Prefer scoped restoration (`local *Package::func = sub { ... }`) so the stub does not leak into subsequent tests.
- A stub that never actually gets called provides no signal. After the code under test runs, assert that the stub was called the expected number of times with the expected arguments.
- `diag()` in mock server handlers or test setup prints the full raw request/response to STDERR on every test run. These are debug artifacts — remove before merge. CI output becomes unreadable when every passing test prints a wall of HTTP text.
- `skip()` called outside a `SKIP:` block causes a test harness warning and may not skip correctly. The correct pattern for conditional module requirements is: `BEGIN { SKIP: { eval { require Module; 1; } or skip($@, 1); } }`. Check that all `skip` calls appear inside a `SKIP:` block with a matching count argument.

---

## References

- **ddclient/ddclient PR #888 review** — namespace mismatch (`CORE::sleep` vs `ddclient::sleep`) and silent float truncation (`sleep(rand($n))`) discovered during real-world review
- **ddclient/ddclient PR #743 review** — `last` vs `next` in host loop, `diag()` debug artifacts, bare heredoc non-interpolation (`<<EoEXAMPLE`), `skip()` outside `SKIP:` block
- **ddclient/ddclient PR #618 review** — deprecated `%$hash{key}` hash slice throughout AWS Signature V4 implementation
- **perldoc perlref** — autovivification and reference safety
- **perldoc perltrap** — common Perl traps including `//` vs `||`, `local` vs `my`
- **perldoc -f open** — three-argument open and injection risk
