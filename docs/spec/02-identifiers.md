# Identifiers

This chapter defines the types that carry names and identifiers in NaN OS records, hashes and signatures with their text forms, the grammars of monikers, principals, paths and origin labels, the identifiers of commits, causes, requests, approvals, plans, cursors, faults, sessions, generations and boots, the time bases, the units, and the canonical rendering of values as text.

## Types

The types of this chapter are items of the library package `ident`, on which every NaN OS package depends. Records of this specification name them without their path.

```nan
// file: ident/package.nan
//! Names, identifiers and times of NaN OS records.

package ident {
    language 1
    targets any
    allow validate in ident
} ident

// file: ident/ident.nan
//! Names, identifiers and times of NaN OS records.

pub closed struct HashValue derives Copy, Eq, Ord, Encode {
    pub bytes: Array[u8, 33] where bytes[0] == 0x01
} HashValue

pub closed struct Signature derives Copy, Eq, Encode {
    pub bytes: Array[u8, 64]
} Signature

pub closed struct Moniker derives Eq, Ord, Encode {
    pub text: text::Fixed[59] where is_moniker(text)
} Moniker

pub closed struct Principal derives Eq, Ord, Encode {
    pub text: text::Fixed[64] where is_principal(text)
} Principal

pub closed struct Path derives Eq, Ord, Encode {
    pub text: text::String where is_path(text)
} Path

pub closed struct OriginLabel derives Eq, Ord, Encode {
    pub text: text::Fixed[260] where is_origin_label(text)
} OriginLabel

pub closed struct RequestId derives Eq, Encode {
    pub text: text::Fixed[32] where is_request_id(text)
} RequestId

pub closed struct CommitId derives Copy, Eq, Ord, Encode {
    pub number: u64
} CommitId

pub closed struct Cause derives Copy, Eq, Ord, Encode {
    pub bits: u64 where bits < 0x8000_0000_0000_0000
} Cause

pub closed struct ApprovalId derives Copy, Eq, Ord, Encode {
    pub number: u64
} ApprovalId

pub closed struct Cursor derives Copy, Eq, Ord, Encode {
    pub seq: u64
} Cursor

pub closed struct SessionId derives Copy, Eq, Ord, Encode {
    pub boot: u64
    pub number: u64
} SessionId

pub closed struct WallTime derives Copy, Eq, Encode {
    pub utc_ns: i64
    pub uncertainty_ns: u64
} WallTime

pub fn is_moniker(text: text::Fixed[59]) -> bool limits pure, notrap, terminates {
    return moniker_ok(text.as_bytes())
} is_moniker

pub fn is_principal(text: text::Fixed[64]) -> bool limits pure, notrap, terminates {
    return principal_ok(text.as_bytes())
} is_principal

pub fn is_path(text: text::String) -> bool limits pure, notrap, terminates {
    let bytes = text.as_bytes()
    if bytes == b"/" {
        return true
    }
    if bytes.len() == 0 or bytes[0] != b'/' {
        return false
    }
    var lex = Lex.Start
    var size: usize = 0
    var dots = true
    var total: usize = 0
    var point: u32 = 0
    var most: u32 = 0
    var digits: usize = 0
    for i in 1..<bytes.len() {
        let byte = bytes[i]
        if lex == Lex.Escaped {
            let Some(c) = char.checked_from_bits(point) else {
                return false
            }
            if point > most or not is_segment_char(c) {
                return false
            }
            size = size.saturating_add(c.utf8_len())
            dots = dots and c == '.'
            lex = Lex.Quoted
        }
        if size > 255 or total > 4_096 {
            return false
        }
        if byte == b'/' and (lex == Lex.Plain or lex == Lex.Closed) and segment_ok(size, dots) {
            total += size + 1
            lex = Lex.Start
            size = 0
            dots = true
        } else if lex == Lex.Start and byte == b'"' {
            lex = Lex.Quoted
        } else if (lex == Lex.Start or lex == Lex.Plain) and is_plain(byte) {
            lex = Lex.Plain
            size += 1
            dots = dots and byte == b'.'
        } else if lex == Lex.Quoted and byte == b'"' {
            lex = Lex.Closed
        } else if lex == Lex.Quoted and byte == b'\\' {
            lex = Lex.Escape
            point = 0
            digits = 0
        } else if lex == Lex.Quoted and raw_byte_ok(bytes, at: i) {
            size += 1
            dots = dots and byte == b'.'
        } else if lex == Lex.Escape and (byte == b'\\' or byte == b'"' or byte == b'\'') {
            lex = Lex.Quoted
            size += 1
            dots = false
        } else if lex == Lex.Escape and byte == b'x' {
            lex = Lex.Hex
            most = 0x7F
        } else if lex == Lex.Escape and byte == b'u' {
            lex = Lex.Brace
            most = 0x10_FFFF
        } else if lex == Lex.Brace and byte == b'{' {
            lex = Lex.Unicode
        } else if lex == Lex.Unicode and byte == b'}' and digits > 0 {
            lex = Lex.Escaped
        } else if (lex == Lex.Hex or lex == Lex.Unicode) and digits < 6 {
            let Some(digit) = hex_value(byte) else {
                return false
            }
            point = point.saturating_mul(16).saturating_add(digit)
            digits += 1
            if lex == Lex.Hex and digits == 2 {
                lex = Lex.Escaped
            }
        } else {
            return false
        }
    }
    return (lex == Lex.Plain or lex == Lex.Closed) and segment_ok(size, dots) and total.saturating_add(size) < 4_096
} is_path

pub fn is_origin_label(text: text::Fixed[260]) -> bool limits pure, notrap, terminates {
    let bytes = text.as_bytes()
    let Some(parts) = bytes.split_at(prefix_length(bytes)) else {
        return false
    }
    return match parts.head {
        b"net." => host_ok(parts.tail)
        b"device." => moniker_ok(parts.tail)
        b"model." => names_ok(parts.tail, extra: b"._-", dotted: false)
        b"principal." => principal_ok(parts.tail)
        b"upload." => session_ok(parts.tail)
        else => false
    }
} is_origin_label

pub fn is_request_id(text: text::Fixed[32]) -> bool limits pure, notrap, terminates {
    for byte in text.as_bytes() {
        if not (is_lower_or_digit(byte) or is_upper(byte) or byte == b'_' or byte == b'-') {
            return false
        }
    }
    return not text.is_empty()
} is_request_id

fn moniker_ok(bytes: Slice[u8]) -> bool {
    var segments: usize = 1
    var run: usize = 0
    for byte in bytes {
        if byte == b'.' and run > 0 and segments < 8 {
            segments += 1
            run = 0
        } else if run < 24 and (is_lower(byte) or (run > 0 and (is_digit(byte) or byte == b'-'))) {
            run += 1
        } else {
            return false
        }
    }
    return bytes.len() <= 59 and run > 0
} moniker_ok

fn principal_ok(bytes: Slice[u8]) -> bool {
    if bytes.len() > 64 {
        return false
    }
    if bytes == b"owner" or bytes == b"system" {
        return true
    }
    let Some(parts) = bytes.split_at(prefix_length(bytes)) else {
        return false
    }
    return match parts.head {
        b"inst." => moniker_ok(parts.tail)
        b"builder." or b"vendor." => names_ok(parts.tail, extra: b"_-@", dotted: false)
        b"human." or b"agent." => names_ok(parts.tail, extra: b"_-@", dotted: true)
        else => false
    }
} principal_ok

fn prefix_length(bytes: Slice[u8]) -> usize {
    for i in 0..<bytes.len() {
        if bytes[i] == b'.' {
            return i + 1
        }
    }
    return 0
} prefix_length

fn names_ok(bytes: Slice[u8], extra: Slice[u8], dotted: bool) -> bool {
    var empty = true
    for byte in bytes {
        if dotted and byte == b'.' and not empty {
            empty = true
        } else if is_lower_or_digit(byte) or (not empty and is_among(byte, set: extra)) {
            empty = false
        } else {
            return false
        }
    }
    return not empty
} names_ok

fn host_ok(bytes: Slice[u8]) -> bool {
    if bytes.len() > 2 and bytes[0] == b'[' and bytes[bytes.len() - 1] == b']' {
        let Some(head) = bytes.prefix(bytes.len() - 1) else {
            return false
        }
        let Some(parts) = head.split_at(1) else {
            return false
        }
        return ipv6_ok(parts.tail)
    }
    return match dotted_numbers(bytes, count: 4) {
        Some(small) => small
        None => dns_name_ok(bytes)
    }
} host_ok

fn dns_name_ok(bytes: Slice[u8]) -> bool {
    var run: usize = 0
    var hyphen = false
    for byte in bytes {
        if byte == b'.' and run > 0 and not hyphen {
            run = 0
        } else if run < 63 and (is_lower_or_digit(byte) or (run > 0 and byte == b'-')) {
            run += 1
            hyphen = byte == b'-'
        } else {
            return false
        }
    }
    return bytes.len() <= 253 and run > 0 and not hyphen
} dns_name_ok

fn dotted_numbers(bytes: Slice[u8], count: usize) -> Option[bool] {
    var numbers: usize = 1
    var empty = true
    var value: u32 = 0
    var small = true
    for byte in bytes {
        if byte == b'.' and not empty and numbers < count {
            numbers += 1
            empty = true
            value = 0
        } else if is_digit(byte) and (empty or value > 0) {
            if value <= 255 {
                value = value * 10 + u32.widen(byte - b'0')
            }
            small = small and value <= 255
            empty = false
        } else {
            return None
        }
    }
    if numbers < count or empty {
        return None
    }
    return Some(small)
} dotted_numbers

fn ipv6_ok(bytes: Slice[u8]) -> bool {
    let groups = ipv6_groups(bytes)
    let Some(parts) = bytes.split_at(7) else {
        return groups.is_some()
    }
    let mapped = parts.head == b"::ffff:"
    return match dotted_numbers(parts.tail, count: 4) {
        Some(small) => mapped and small
        None => groups.is_some() and not (mapped and groups == Some(3))
    }
} ipv6_ok

fn ipv6_groups(bytes: Slice[u8]) -> Option[usize] {
    var groups: usize = 0
    var digits: usize = 0
    var colons: usize = 0
    var gap = false
    var zero = false
    var run: usize = 0
    var before: usize = 0
    var after: usize = 0
    for byte in bytes {
        if byte == b':' and colons == 0 {
            colons = 1
            digits = 0
        } else if byte == b':' and colons == 1 and not gap and run == 0 {
            colons = 2
            gap = true
        } else if is_lower_hex(byte) and digits == 0 {
            if groups >= 8 or (colons == 1 and groups == 0) or (colons == 2 and byte == b'0') {
                return None
            }
            groups += 1
            digits = 1
            colons = 0
            zero = byte == b'0'
            if zero {
                run = run.saturating_add(1)
            } else {
                run = 0
            }
            if gap and run > after {
                after = run
            }
            if not gap and run > before {
                before = run
            }
        } else if is_lower_hex(byte) and digits > 0 and digits < 4 and not zero {
            digits += 1
        } else {
            return None
        }
    }
    if colons == 1 {
        return None
    }
    if gap and groups <= 6 and before < 8 - groups and after <= 8 - groups {
        return Some(groups)
    }
    if not gap and groups == 8 and before < 2 {
        return Some(groups)
    }
    return None
} ipv6_groups

fn session_ok(bytes: Slice[u8]) -> bool {
    let Some(parts) = bytes.split_at(1) else {
        return false
    }
    return parts.head == b"s" and dotted_numbers(parts.tail, count: 2).is_some()
} session_ok

enum Lex derives Copy, Eq {
    Start
    Plain
    Quoted
    Escape
    Hex
    Brace
    Unicode
    Escaped
    Closed
} Lex

fn segment_ok(size: usize, dots: bool) -> bool = size > 0 and size <= 255 and not (dots and size <= 2)

fn raw_byte_ok(bytes: Slice[u8], at: usize) -> bool {
    if at < bytes.len() and bytes[at] >= 0x80 and bytes[at] < 0xC0 {
        return true
    }
    let Some(c) = text::utf8::char_at(bytes, at) else {
        return false
    }
    return is_segment_char(c)
} raw_byte_ok

fn is_segment_char(c: char) -> bool {
    return match c {
        '\u{0}'..='\u{1F}' or '/' or '\u{7F}'..='\u{A0}' or '\u{AD}' or '\u{34F}' or '\u{61C}' or
            '\u{115F}'..='\u{1160}' or '\u{1680}' or '\u{17B4}'..='\u{17B5}' or '\u{180B}'..='\u{180F}' or
            '\u{2000}'..='\u{200F}' or '\u{2028}'..='\u{202F}' or '\u{205F}'..='\u{206F}' or '\u{3000}' or
            '\u{3164}' or '\u{FE00}'..='\u{FE0F}' or '\u{FEFF}' or '\u{FFA0}' or '\u{FFF0}'..='\u{FFFB}' or
            '\u{1BCA0}'..='\u{1BCA3}' or '\u{1D173}'..='\u{1D17A}' or '\u{E0000}'..='\u{E0FFF}' => false
        else => true
    }
} is_segment_char

fn hex_value(byte: u8) -> Option[u32] {
    if is_digit(byte) {
        return Some(u32.widen(byte - b'0'))
    }
    if byte >= b'a' and byte <= b'f' {
        return Some(u32.widen(byte - b'a') + 10)
    }
    if byte >= b'A' and byte <= b'F' {
        return Some(u32.widen(byte - b'A') + 10)
    }
    return None
} hex_value

fn is_among(byte: u8, set: Slice[u8]) -> bool {
    for member in set {
        if member == byte {
            return true
        }
    }
    return false
} is_among

fn is_plain(byte: u8) -> bool = is_lower_or_digit(byte) or is_upper(byte) or is_among(byte, set: b"._@+=-")

fn is_lower_hex(byte: u8) -> bool = is_digit(byte) or (byte >= b'a' and byte <= b'f')

fn is_lower_or_digit(byte: u8) -> bool = is_lower(byte) or is_digit(byte)

fn is_lower(byte: u8) -> bool = byte >= b'a' and byte <= b'z'

fn is_upper(byte: u8) -> bool = byte >= b'A' and byte <= b'Z'

fn is_digit(byte: u8) -> bool = byte >= b'0' and byte <= b'9'
```

- The `text` of a `Moniker`, `Principal`, `Path`, `OriginLabel` or `RequestId` is exactly a text the grammar and rules of its section below admit, and its decoder rejects every other text.
- A `HashValue` whose leading byte is not `0x01` is no hash and fails to decode.

## Hashes

Every hash NaN OS computes or stores is a NaN hash value ([NaN 29 §Hash values](https://github.com/nan-language/nan/blob/main/docs/spec/29-identity-versioning.md#hash-values)): 33 bytes, the tag `0x01` then a BLAKE3-256 digest, held in a `HashValue`.

NaN OS hashes an object of kind K as the hash of the UTF-8 bytes of K and LF, followed by the object's bytes as its home chapter defines them. A kind is a lower-case word naming one sort of object, never one of NaN's own (`item`, `interface`, `instance`, `type`, `closure`, `variants`). Content is hashed alone, its digest being the standard BLAKE3-256 of the content: a blob ([11-store](11-store.md#identity)) and each file a manifest names ([03-boot](03-boot.md#generations-and-manifests)).

| Kinds | Objects | Home |
| --- | --- | --- |
| `record`, `secret`, `stream`, `ephemeral`, `dir`, `entry`, `run`, `transaction` | nodes, dir digests and transactions | [11-store](11-store.md#identity) |
| `manifest` | generation manifests | [03-boot](03-boot.md#generations-and-manifests) |
| `code`, `bundle`, `review`, `seal`, `certificate` | bundles and their signed parts | [10-code](10-code.md#artefacts-and-bundles) |
| `plan`, `read-set` | plans and their read sets | [14-changes](14-changes.md#the-plan-hash) |
| `approval`, `policy` | approval decisions and the policy | [19-oversight](19-oversight.md#approvals) |
| `request` | signed requests: the owner key and loader requests of the boot record, and requests sent outside a session | [03-boot](03-boot.md#claim-at-boot), [19-oversight](19-oversight.md#emergency-controls) |
| `audit`, `anchor` | audit entries and anchors | [17-observability](17-observability.md#audit-chain) |
| `handshake` | session challenges, connection bindings and Noise handshake hashes | [18-link](18-link.md#connections-and-sessions), [16-network](16-network.md#network-transports-of-the-link) |

Records hashed under one kind differ in their field keys, so their `mem::wire` encodings never coincide.

A **signature** over an object is the Ed25519 signature (RFC 8032, pure Ed25519, no context) of the object's 33-byte hash, held in a `Signature`. The object's kind is the signature's **purpose**, so a signature over one kind of object is invalid for every other kind; [signatures](09-authority.md#signatures) lists each purpose's signers.

## Text forms of binary values

- A response writes a hash in NaN's short form (18 characters), or in its full form (55 characters) when the request asks for full forms ([18-link](18-link.md#responses)). Every text format holds full forms; records, files and signed bytes hold the 33 bytes.
- A request may write a hash in either form. A short form denotes the hash whose short form it is, among the hashes of the kind its operand expects that the session's capabilities reach. A short form matching none or several of them fails as an unknown or an ambiguous name ([18-link](18-link.md#errors-and-fixes)), the error listing the full forms that match.
- No signature, approval, record or file holds a short form.
- A signature (64 bytes) and a public key (32 bytes) are written as NaN's checksummed base32 of their bytes, 105 and 54 characters long.

## Names

Each kind of name has the grammar of its section, in NaN's EBNF dialect ([NaN 00 §Grammar notation](https://github.com/nan-language/nan/blob/main/docs/spec/00-charter.md#grammar-notation)).

### Monikers

A **moniker** names an [instance](06-execution.md#instances).

```ebnf
MONIKER = MONIKER_SEGMENT ( "." MONIKER_SEGMENT )* ;
MONIKER_SEGMENT = "a" .. "z" ( "a" .. "z" | "0" .. "9" | "-" )* ;
```

- A moniker has 1 to 8 segments of at most 24 characters each and at most 59 bytes in all, so `inst.` followed by a moniker is a principal.
- The instance of a moniker with two or more segments is supervised by the instance its text before the last `.` names; the instance of a one-segment moniker is supervised by `supervisor` ([20-base](20-base.md#supervisor)).
- The one-segment moniker of a [base instance](20-base.md) names that instance only, and the moniker `auth` names no instance ([instances](06-execution.md#instances)).
- A moniker never holds `/` and a path always begins with `/`, so no text is both.

`devices.ahci0` is supervised by `devices`, holds grants as the principal `inst.devices.ahci0` and keeps its [dead letters](06-execution.md#dead-letters) in `/state/devices.ahci0/dead`.

### Principals

The texts naming [principals](09-authority.md#principals) are a subset of NaN's principal form ([NaN 31 §Seals](https://github.com/nan-language/nan/blob/main/docs/spec/31-trust.md#seals)):

```ebnf
PRINCIPAL = "owner" | "system" | ( "human" | "agent" ) "." NAME ( "." NAME )*
    | ( "builder" | "vendor" ) "." NAME | "inst." MONIKER ;
NAME = ( "a" .. "z" | "0" .. "9" ) ( "a" .. "z" | "0" .. "9" | "_" | "-" | "@" )* ;
```

A principal holds at most 64 bytes, and principals compare by bytes. Its kind follows from its text:

| Text | Kind |
| --- | --- |
| `owner` | owner |
| `system` | system |
| `human.<name>` | human |
| `agent.<name>` | agent |
| `builder.<name>` | builder |
| `vendor.<name>` | vendor |
| `inst.<moniker>` | instance |
| `human.<name>` or `agent.<name>`, then one or more `.<name>` | child |

A child principal's parent is the principal its text names without its last segment ([09-authority](09-authority.md#delegation)); `owner` has no children. A child is never of kind human or agent, whatever its prefix: `agent.ops.triage` is a child of the agent `agent.ops`, and `human.ana.helper` is no human.

### Paths

A path names a node of the [tree](13-tree.md#path-resolution).

```ebnf
PATH = "/" | ( "/" PATH_SEGMENT )+ ;
PATH_SEGMENT = PLAIN_SEGMENT | TEXT ;
PLAIN_SEGMENT = PLAIN_RUN - ( "." | ".." ) ;
PLAIN_RUN = PLAIN_CHAR+ ;
PLAIN_CHAR = "A" .. "Z" | "a" .. "z" | "0" .. "9" | "." | "_" | "@" | "+" | "=" | "-" ;
```

- `TEXT` is a NaN text literal ([NaN 01 §Text literals](https://github.com/nan-language/nan/blob/main/docs/spec/01-lexical.md#text-literals)), whose value is the segment.
- A segment is 1 to 255 bytes of UTF-8 holding no `/`, no code point below U+0020 or from U+007F to U+009F, and no code point NaN rejects in source ([NaN 01 §Rejected code points](https://github.com/nan-language/nan/blob/main/docs/spec/01-lexical.md#rejected-code-points)); it is never `.` or `..`.
- A path holds at most 4,096 bytes, counting its segments' bytes and one byte per `/`.
- Segments compare by bytes, without Unicode normalisation.
- The canonical rendering writes a segment as a `PLAIN_SEGMENT` when it is one, and otherwise as a text literal. A request may write any segment as a text literal.

`/data/agent.ops/"notes 2026.txt"` names the node `notes 2026.txt` in the dir `/data/agent.ops`.

### Origin labels

An **origin label** names where untrusted content came from ([19-oversight](19-oversight.md#taint)).

```ebnf
ORIGIN_LABEL = "net." HOST | "device." MONIKER | "model." MODEL_NAME | "principal." PRINCIPAL | "upload.s" NUMBER "." NUMBER ;
HOST = DNS_NAME | IPV4 | "[" IPV6 "]" ;
DNS_NAME = DNS_LABEL ( "." DNS_LABEL )* ;
DNS_LABEL = ( "a" .. "z" | "0" .. "9" ) ( ( "a" .. "z" | "0" .. "9" | "-" )* ( "a" .. "z" | "0" .. "9" ) )? ;
IPV4 = NUMBER "." NUMBER "." NUMBER "." NUMBER ;
IPV6 = ( "0" .. "9" | "a" .. "f" | ":" | "." )+ ;
MODEL_NAME = ( "a" .. "z" | "0" .. "9" ) ( "a" .. "z" | "0" .. "9" | "." | "_" | "-" )* ;
```

- A host matching `IPV4` is an IPv4 address whose numbers are at most 255. A `DNS_NAME` has at most 253 bytes and labels of at most 63. An `IPV6` is an IPv6 address in the text form of RFC 5952.
- The session of an `upload.` label is written with its boot, as `s<b>.<n>`.
- A label holds at most 260 bytes.

| Label | Content it marks |
| --- | --- |
| `net.<host>` | data received from a network peer ([16-network](16-network.md#egress-gate)) |
| `device.<moniker>` | data a device delivered through the driver instance named |
| `model.<name>` | output of a model run by `infer` ([20-base](20-base.md#infer)) |
| `principal.<principal>` | text another principal wrote |
| `upload.<session>` | bytes a session uploaded with `put` ([18-link](18-link.md#verbs)) |

## Identifiers

| Identifier | Text | Type | Unique within | Assigned by |
| --- | --- | --- | --- | --- |
| commit | `#<n>` | `CommitId` | a store, across its branches | the store, in commit order from `#1` ([11-store](11-store.md#transactions)) |
| cause | `c<b>.<n>` | `Cause` | a machine | the core ([17-observability](17-observability.md#causes)) |
| request | 1 to 32 letters, digits, `_` and `-` | `RequestId` | a connection's pending requests | the client ([18-link](18-link.md#requests)) |
| approval | `a<n>` | `ApprovalId` | a store | `policy` ([19-oversight](19-oversight.md#approvals)) |
| plan | the short form of its plan hash | `HashValue` | the plans a session previewed | [14-changes](14-changes.md#preview) |
| cursor | `e<n>` | `Cursor` | a stream | the stream's writer ([11-store](11-store.md#streams)) |
| fault | `f<n>` | `u64` | a machine | `journal`: the sequence number of the fault's record in `/journal/faults` ([17-observability](17-observability.md#the-journal-and-its-streams)) |
| session | `s<n>` or `s<b>.<n>` | `SessionId` | a machine | `link` ([18-link](18-link.md#connections-and-sessions)) |
| generation | `<n>` | `u64` | a machine | its manifest ([03-boot](03-boot.md#generations-and-manifests)) |
| boot | `<n>` | `u64` | a machine | the loader ([03-boot](03-boot.md#boot-record)) |

```ebnf
COMMIT_ID = "#" NUMBER ;
CAUSE_ID = "c" NUMBER "." NUMBER ;
REQUEST_ID = ( "A" .. "Z" | "a" .. "z" | "0" .. "9" | "_" | "-" )+ ;
APPROVAL_ID = "a" NUMBER ;
CURSOR = "e" NUMBER ;
FAULT_ID = "f" NUMBER ;
SESSION_ID = "s" NUMBER ( "." NUMBER )? ;
NUMBER = "0" | "1" .. "9" ( "0" .. "9" )* ;
```

- A `Cause` holds b, the boot counter modulo 2^20, in bits 62 to 43, and n in bits 42 to 0; its bit 63 is 0. n counts from 1 in each boot and wraps from 2^43 − 1 to 1. A message header carries a cause in its low 63 bits, its bit 63 marking the cause traced ([07-communication](07-communication.md#headers)).
- A `SessionId` holds the boot counter b of the boot the session opened in and n, counting from 1 in each boot. `s<n>` names a session of the current boot and `s<b>.<n>` any session. A session never outlives its boot.
- A cursor `e<n>` names the record of sequence number n of a stream; a watch from `e<n>` delivers the records after it ([18-link](18-link.md#pushed-records)).
- A generation number is at least 1; generation 0 names the development boot ([03-boot](03-boot.md#development-loader)).

The commit `#1` of every store is its genesis commit, and `c3.12` is the twelfth cause of the machine's third boot.

## Time

- **Monotonic time** is a `u64` count of nanoseconds since the kernel's entry function started in the current boot (requires [N9](22-nan.md#n9)). It never decreases within a boot and restarts at each boot ([08-scheduling](08-scheduling.md#clocks)). A record kept beyond its boot holds a monotonic time only beside the boot counter or a cause, which holds it.
- **Wall time** is a `WallTime`, which `clock` provides ([08-scheduling](08-scheduling.md#clocks)). `utc_ns` counts nanoseconds since 1970-01-01T00:00:00Z, every day being 86,400 s long, so leap seconds are not counted. `uncertainty_ns` bounds the difference between `utc_ns` and UTC; a value of 2^63 or more means unknown.
- A wall time renders as the RFC 3339 date and time in UTC with milliseconds rounded down, `Z`, `~`, and the uncertainty rounded up to whole milliseconds with unit `ms` when below 10 s, else to whole seconds with unit `s`, or `unknown`. A request may write a wall time with 0 to 9 fractional digits and may omit `~` and the uncertainty, which is then 0.

```text
2026-10-29T21:14:03.120Z~12ms
```

## Units

| Quantity | Field suffix | Units in text |
| --- | --- | --- |
| size | `_bytes` | B, KiB, MiB, GiB, TiB: 1, 2^10, 2^20, 2^30, 2^40 bytes |
| duration | `_ns`, `_us`, `_ms`, `_s`, `_min`, `_h` | ns, µs, ms, s, min, h |
| energy | `_mj` | mJ |
| power | `_mw` | mW |
| frequency | `_hz`, `_mhz` | Hz, MHz, GHz |
| fraction | `_pct` | % |
| output | `_units` | units |

- A record field holding a quantity is an integer in the unit its name's suffix gives or its definition states; without either, a size is in bytes and a monotonic time or a duration in nanoseconds.
- An output unit is 4 bytes of rendered output: a response's units are its bytes divided by 4, rounded up ([18-link](18-link.md#budgets)).
- This specification writes sizes with binary prefixes and durations with the units of the table.

## Canonical rendering

The **canonical rendering** of a value is the text the link writes for it in records ([18-link](18-link.md#responses)) and decodes from operands of its type. It depends only on the value, its type and the hash form the request asks for, so equal values render to equal bytes on every machine and in every run.

| Type | Rendering |
| --- | --- |
| integer | decimal digits, with `-` before a negative value |
| `bool` | `true` or `false` |
| `f32`, `f64` | as [NaN 13 §Render](https://github.com/nan-language/nan/blob/main/docs/spec/13-text-encode.md#render) writes them |
| `char` | a char literal, escaped as in a text literal |
| `text::String`, `text::Fixed[N]` | a text literal: `"`, the characters, `"`, with `\"`, `\\`, `\n`, `\r`, `\t` and `\0` for those characters and `\u{H}`, in uppercase hexadecimal without leading zeros, for every other code point below U+0020 or from U+007F to U+009F and every code point NaN rejects in source |
| `mem::Bytes` | as NaN 13 §Render writes it: `b"`, the bytes, `"` |
| struct | its name, `(`, each field as `name: value` in declaration order separated by `, `, then `)` |
| enum | `Name.Variant`, followed by `(value)` for a payload or `(label: value, ...)` for labelled payloads |
| `Option`, `Result` | `None`, `Some(v)`, `Ok(v)`, `Err(e)` |
| `Array`, `mem::List`, `mem::Bounded` | `[`, the elements separated by `, `, then `]` |
| `mem::Set` | as a list, in ascending order |
| `mem::Map` | `{`, each entry as `key: value` in ascending key order separated by `, `, then `}` |
| `mem::Box`, `mem::Shared` | the contents |
| `HashValue` | its short or full form ([Text forms of binary values](#text-forms-of-binary-values)) |
| `Signature` | its checksummed base32 |
| `Moniker`, `Principal`, `OriginLabel`, `RequestId` | its text, without quotes |
| `Path` | its path text ([Paths](#paths)) |
| `CommitId`, `Cause`, `ApprovalId`, `Cursor`, `SessionId` | its text form ([Identifiers](#identifiers)) |
| `WallTime` | its text form ([Time](#time)) |
| `Untrusted[T]` | an envelope ([18-link](18-link.md#responses)) |

- A type's name is the last segment of its canonical path; generic arguments never appear.
- A record of the tree renders by the type its node declares ([11-store](11-store.md#declared-types-and-validity)).
- An operand written in the canonical rendering of its type decodes to the value it renders (requires [N20](22-nan.md#n20)). The link renders and decodes values of types it does not hold through the type descriptions and decoders the declaring bundle exports (requires [N20](22-nan.md#n20)).
- In JSON output ([18-link](18-link.md#responses)), a value of a type of [Types](#types) is a JSON string holding its rendering.

```text
Manifest(generation: 7, kernel: ahocrmymkorvfrrdcb, genesis: ag52ibmj4vmfqiamgr, min_loader: 3, store_format: 1, signer: vendor.nanos)
```
