# capa_validate

Pure-Capa **input validation**: composable, structural (shape-level)
checks over untrusted strings and integers. Zero capabilities: every
validator is a `(String) -> Result<...>` or `(Int) -> Result<...>`
function. Nothing here can touch the filesystem, the network, the clock,
randomness, or anything else; the library holds no authority and reads no
global state. `capa --manifest` proves it (see
[Audit claim](#audit-claim)). Output is byte-identical on the Python and
Wasm backends.

These are **structural checks to reject malformed input early**, not
security guarantees and not authoritative parsers. The email and URL
checks are **heuristics**. Read [Honest posture](#honest-posture) before
using any of them for a trust decision.

## Status

v0.1 (seed library). **Zero runtime dependencies** by design (`capa_test`
is a dev-dependency only): every check is a few lines of pure Capa, so
pulling in another library would add supply-chain surface for no real
simplification. Scope:

- **String shape:** `non_empty`, `trimmed_non_empty`, `length_between`,
  `is_ascii`, `is_digits`, `is_alphanumeric`, `matches_charset`,
  `one_of`.
- **Numeric:** `int_in_range`, `parse_int_in_range` (parse **and**
  bound), `is_integer_string`, `is_decimal_string`.
- **Heuristic structural checks (non-authoritative):**
  `looks_like_email`, `looks_like_url`.
- **Errors:** a `ValidationError` sum type with an `error_message`
  helper, plus a `Form` collector (`check_field` / `is_valid` /
  `messages`) for validating several named fields at once and reporting
  every failure, not just the first.

## Quick start

```capa
import capa_validate.validate

fun main(stdio: Stdio)
    // One field at a time. Every validator returns the validated value
    // in `Ok`, so checks compose and read left to right.
    match length_between("hunter2", 6, 64)
        Ok(s)  -> stdio.println("password shape ok: ${s}")
        Err(e) -> stdio.println(error_message(e))

    // Parse AND bound an untrusted integer in one step.
    match parse_int_in_range("42", 0, 100)
        Ok(n)  -> stdio.println("n = ${n}")
        Err(e) -> stdio.println(error_message(e))

    // A whole form: collect every failure at once.
    let form = new_form()
    check_field(form, "email", looks_like_email("not-an-email"))
    check_field(form, "age", parse_int_in_range("200", 0, 130))
    if not form.is_valid()
        for line in form.messages()
            stdio.println(line)
    // -> email: not a structurally valid email address: must contain exactly one '@'
    // -> age: must be in the range 0..130
```

The full runnable example is [`example.capa`](./example.capa).

```bash
capa --run example.capa
capa --wasm --run example.capa   # byte-identical output
```

## Install via capa.toml

```toml
[dependencies.capa_validate]
git = "https://github.com/nelsonduarte/capa_validate"
tag = "v0.1.0"
verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B"
```

`capa install` runs `git verify-tag` against your GPG keyring; import
the publisher's key first (see [`SECURITY.md`](SECURITY.md) for the
fingerprint provenance and `gpg --import` instructions).

## API surface

All validators live in `capa_validate.validate`. The `is_*` string / set
checks return `Result<String, ValidationError>`, carrying the validated
value through on `Ok` so checks chain with `?`. The two `is_*_string`
predicates return a `Bool` (they answer a yes/no shape question).

```capa
// String shape.
pub fun non_empty(s: String)                              -> Result<String, ValidationError>
pub fun trimmed_non_empty(s: String)                      -> Result<String, ValidationError>  // returns the TRIMMED value
pub fun length_between(s: String, min: Int, max: Int)     -> Result<String, ValidationError>  // inclusive, counts code points
pub fun is_ascii(s: String)                               -> Result<String, ValidationError>  // reports first non-ASCII index
pub fun is_digits(s: String)                              -> Result<String, ValidationError>  // non-empty, all 0-9
pub fun is_alphanumeric(s: String)                        -> Result<String, ValidationError>  // non-empty, all A-Za-z0-9
pub fun matches_charset(s: String, allowed: String)       -> Result<String, ValidationError>  // every char in `allowed`
pub fun one_of(s: String, options: List<String>)          -> Result<String, ValidationError>  // exact, case-sensitive

// Numeric.
pub fun int_in_range(n: Int, lo: Int, hi: Int)            -> Result<Int, ValidationError>      // inclusive
pub fun parse_int_in_range(s: String, lo: Int, hi: Int)   -> Result<Int, ValidationError>      // parse AND bound
pub fun is_integer_string(s: String)                      -> Bool                              // syntactic
pub fun is_decimal_string(s: String)                      -> Bool                              // syntactic

// Heuristic structural checks (NON-AUTHORITATIVE, see Honest posture).
pub fun looks_like_email(s: String)                       -> Result<String, ValidationError>
pub fun looks_like_url(s: String)                         -> Result<String, ValidationError>

// Errors and form-style validation.
pub fun error_message(e: ValidationError)                 -> String
pub fun new_form()                                        -> Form
pub fun check_field<T>(form: Form, name: String, result: Result<T, ValidationError>)
pub fun field_error_message(fe: FieldError)               -> String
// Form methods: is_valid() -> Bool, error_count() -> Int, messages() -> List<String>
```

### `ValidationError`

```capa
pub type ValidationError =
    Empty                    // a value required to be non-empty was empty
    TooShort(Int, Int)       // (length, minimum)
    TooLong(Int, Int)        // (length, maximum)
    NotAscii(Int)            // code-point index of the first non-ASCII character
    NotDigits
    NotAlphanumeric
    NotInSet(String)         // the value, which is not one of the options
    DisallowedChar(String)   // a character outside the allowed charset
    NotInteger               // not a base-10 integer within the i64 range
    NotDecimal
    OutOfRange(Int, Int)     // (lo, hi)
    NotEmail(String)         // reason
    NotUrl(String)           // reason
```

`error_message` renders any variant as a human-readable string.
`check_field` appends a `FieldError { field, error }` to a `Form` for
each failed field; `Form.messages()` renders them as
`"<field>: <message>"` lines, in the order the fields were checked.

### Composing several checks

`?` propagates the first failure; a `Form` collects them all:

```capa
// Fail fast at the first failure.
fun check_handle(s: String) -> Result<String, ValidationError>
    let trimmed = trimmed_non_empty(s)?
    let sized = length_between(trimmed, 3, 20)?
    return is_alphanumeric(sized)

// Or collect every field's failure for a form.
let form = new_form()
check_field(form, "handle", check_handle(input_handle))
check_field(form, "port", parse_int_in_range(input_port, 1, 65535))
```

## The i64 note (parsing large numbers)

Capa's `Int` is a signed 64-bit integer. The **Wasm backend traps on
i64 overflow**, while the Python backend uses arbitrary-precision
integers. To keep the two backends identical, `parse_int_in_range`
delegates parsing to the built-in `parse_int`, which **fails closed to
`None`** for any string whose magnitude falls outside the signed 64-bit
range `[-2^63, 2^63)`. A too-large numeric string is therefore rejected
as `NotInteger`, **identically on both backends**, and is never
converted to a value that would trap:

```capa
parse_int_in_range("9223372036854775807", 0, 9223372036854775807)  // Ok(9223372036854775807)  (i64 max)
parse_int_in_range("9223372036854775808", 0, 9223372036854775807)  // Err(NotInteger)          (i64 max + 1)
parse_int_in_range("99999999999999999999999999999999", 0, 100)     // Err(NotInteger)          (far out of range)
```

`is_integer_string` answers a different, purely **syntactic** question
(optional sign then digits) and returns `true` for a magnitude that
`parse_int_in_range` rejects; use `parse_int_in_range` when you actually
need the `Int`.

## Verification

`parse_int_in_range` and the digit / decimal classification were checked
**oracle-first**: a small Python reference (kept **outside** the
repository) mirrors the exact semantics of the built-in `parse_int`
(decimal only, optional sign, surrounding ASCII whitespace, the
`[-2^63, 2^63)` window) and the digit classification. The Capa suites
assert against those expected values on both backends, including the
i64-boundary cases (i64 max, i64 max + 1, i64 min, i64 min - 1, and a
32-digit magnitude), which prove the clean, identical rejection described
above.

The email / URL suites pin the documented **heuristic** behaviour against
adversarial inputs: an empty local part, no dot, multiple `@`, embedded
spaces, a missing scheme, an empty host, and (importantly) a nonsense
address of the right shape that the heuristic **accepts**, because it
cannot tell.

```bash
capa test          # Python backend
capa test --both   # Python + Wasm, byte-identical stdout required
```

Current output of `capa test --both`:

```
capa test: 4 file(s) under .../capa_validate/tests [backend: python+wasm]
test_form.capa ... ok
test_heuristics.capa ... ok
test_numeric.capa ... ok
test_strings.capa ... ok
4 test(s): 4 passed, 0 failed
```

`capa_test` is declared under `[dev-dependencies]` with the same
git + tag + verify_key shape as any published dependency, pinned to its
`v0.1.0` tag and verified against the publisher key, so `capa install`
runs the full three-layer check (lockfile SHA + GPG tag signature +
SLSA L2 provenance) on it. Dev-dependencies are resolved only when this
repository is the install root, so a consumer of `capa_validate` never
fetches the test library.

## Audit claim

`capa --manifest` over `validate.capa` reports, for every function:

```
declared_capabilities:                []
transitively_reachable_capabilities:  []
has_unsafe:                           false
user_defined_capabilities:            []
```

0 functions with capabilities, 0 crossing `unsafe`. The only capability
anywhere in this repository is in the example and is the example's own
(`Stdio`, to print). A program using `capa_validate` declares only the
authority its own code needs.

## Honest posture

- **Structural, not semantic.** These checks catch malformed input
  early. They do not sanitise, escape, or make a value safe to use in
  SQL, HTML, a shell, a path, or any other sink. Escaping is the sink's
  job; validate the shape here, then escape at the boundary.
- **The email / URL checks are heuristics.** `looks_like_email` verifies
  a shape (no whitespace, exactly one `@`, a non-empty local part, and a
  domain with an interior dot). It is **not** RFC 5322, does **not**
  prove the address exists or is deliverable, and **accepts** many
  strings a real mail system would reject (for example `....@x.y`). It
  also **rejects** some technically valid-but-exotic addresses (quoted
  local parts, IP-literal domains). The whitespace check rejects only the
  four ASCII whitespace characters (space, tab, newline, carriage
  return); it does **not** reject other ASCII control characters or an
  embedded NUL, so `"a\u{1}b@c.d"` passes and a value that passes is
  **not** guaranteed to be printable. `looks_like_url` checks for an
  `http(s)://` scheme and a non-empty host; it does **not** validate the
  URL per RFC 3986, confirm the host resolves, or judge whether the URL
  is safe to fetch, and it shares the same loose whitespace rule (a host
  containing a control character or NUL is accepted). Use both only to
  reject obvious garbage early, then rely on a real parser and, for URLs,
  an SSRF-aware allow-list before making any request.
- **`parse_int_in_range` accepts surrounding whitespace and a sign.**
  It delegates to the built-in `parse_int` (`"  -7 "` parses to `-7`).
  If you need to reject whitespace, run `is_integer_string` first or
  `trim` yourself. It rejects underscores, `0x`/`0b`/`0o` bases, and any
  out-of-i64 magnitude (see the i64 note).
- **`is_digits` / `is_alphanumeric` are ASCII-only.** They accept only
  the ASCII code points `0-9` (and, for `is_alphanumeric`, `A-Za-z`). A
  Unicode digit such as `٣` (Arabic-Indic three) or `３` (fullwidth
  three) is **rejected** as `NotDigits` / `NotAlphanumeric`. The same
  applies to the `DIGITS` used by `is_decimal_string`.
- **`is_integer_string` / `is_decimal_string` are syntactic.** They
  check character shape only. `is_integer_string` returns `true` for a
  magnitude too large for an `Int`; `is_decimal_string` accepts `".5"`
  and `"5."` and makes no precision or range promise. Parse with
  `parse_int_in_range` when you need the value.

## License

MIT. See [`LICENSE`](./LICENSE). Release tags are GPG-signed; see
[`SECURITY.md`](./SECURITY.md) for the fingerprint and verification
instructions.
