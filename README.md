# textproto

Pure GoML framing for line-oriented text protocols. The package keeps protocol
bytes separate from character decoding so callers can apply HTTP or mail policy
without silently replacing non-UTF-8 octets.

`Reader[R: Read]` owns a buffered source. `read_line` returns one physical line
without its CRLF, or `None` only at clean EOF. Bare CR/LF and unterminated lines
fail. `read_headers(allow_folded)` reads a blank-line-terminated block, keeps
duplicate fields in order and canonicalizes ASCII names. Folded continuation
lines are rejected unless explicitly enabled; enabled folds join with one space.
Values preserve non-ASCII bytes, while `Field::value_text` requests UTF-8
validation. `parse_header_line` exposes the same single-field validation to
protocol adapters that own their own transport buffering.

`Headers::set(name, value)` validates and copies a value, replaces all matching
fields at the position of the first match, or appends when absent.
`Headers::remove(name)` removes every matching field and returns the count.
Both use the same case-insensitive ASCII name policy as `append`; invalid names
or values leave the collection unchanged. Unrelated fields retain their order.
Header aliases share these mutations, while `fields()` and `values()` return
independent byte snapshots. These in-memory methods do not impose wire limits;
`Writer::write_headers` applies its configured bounds before output.

`read_dot_chunk` incrementally removes dot stuffing and recognizes a line
containing only a dot as the terminator. `read_dot_bytes` is a bounded convenience
that materializes the decoded block. `Writer[W: Write]` writes CRLF lines and
header blocks, then `begin_dot`, `write_dot_chunk` and `finish_dot` frame a dot
block. Input chunks can split CRLF. A final unterminated data line gets CRLF
before the terminator. Writer I/O errors and reader parse/source errors are
terminal; invalid writer input is checked before emitting that chunk.

`Limits` independently bounds physical line content, total header wire bytes,
field count, unfolded field value bytes and decoded dot-body bytes. The physical
line limit includes the extra dot used for stuffing and the one-byte terminator,
but excludes CRLF. Reader and writer apply the same bound. Those limits
are application policy, not SMTP's fixed transport line limit. Callers needing
SMTP's 1000-octet line rule should set `max_line_bytes` accordingly. Header
values reject control bytes other than horizontal tab but can contain arbitrary
high bytes; consumers must choose a character policy explicitly. `read_response(max_lines, max_bytes)` reads a numeric reply with a 100–599
code. A hyphen continues a reply; a space or a bare three-digit code ends it.
Every continuation must repeat the code. The returned `Reply` keeps each text
line as bytes, including empty lines and non-UTF-8 octets. It consumes exactly
one reply; the next command or reply remains buffered. This supports
[SMTP multiline replies](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.2)
without interpreting success or protocol-specific code ranges. It does not accept FTP's unprefixed intermediate lines.

`EnhancedStatus::parse` validates [RFC 3463](https://www.rfc-editor.org/rfc/rfc3463.html)
mail status codes such as `5.1.1`: class 2/4/5, one to three decimal digits per
subject/detail, and no leading zeroes. Unknown subject/detail numbers within
those bounds are preserved for extension compatibility. The public fields are
`class`, `subject`, and `detail`; `to_string()` gives the dotted representation.
`Reply.enhanced_status()` optionally inspects the first token of each text line:
all lines must omit it or carry the same code, whose class must match the numeric
reply. A digit followed by a dot marks a candidate; malformed candidates fail.
This follows [RFC 2034](https://www.rfc-editor.org/rfc/rfc2034.html)'s multiline
consistency rule. Inspection does not mutate the reply/reader or interpret the
human-readable remainder; callers decide whether to require enhanced codes.

Reply budgets count all physical lines and wire bytes, including codes and CRLF,
in addition to `max_line_bytes`. Malformed, truncated and over-budget replies
make the reader terminal. Invalid limit arguments and an active dot block fail
without consuming input. The package does not parse MIME encoded words, HTTP
start lines or multipart bodies.

`Writer::write_response(reply, max_lines, max_bytes)` writes the corresponding
numeric reply format. Codes must be 100–599 and `Reply.lines` must contain at
least one byte string. Continuation lines use `code-`, the final nonempty line
uses `code `, and an empty final line uses the bare code. It preserves high bytes
and horizontal tabs, rejecting other control bytes, CR and LF. The per-line
limit includes the code and separator; the whole-reply budget also includes
CRLF. The complete reply is validated and buffered within that budget before
the sink receives any bytes. Invalid data or limits leave the writer usable;
an I/O failure is terminal and may have written a prefix. Writing during an
active dot block is rejected. Enhanced status consistency remains an explicit
`Reply.enhanced_status()` check chosen by the application.

Run `(cd ../verification && just ecosystem-test textproto)` from this library repository.

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest and its dependencies. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test textproto)` also retains the library-specific smoke and compatibility checks.
