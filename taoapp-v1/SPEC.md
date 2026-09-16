# TAOAppv1: structured subnet content in the on-chain identity

Status: v1, frozen. Published 2026-09-10.

Bittensor subnet owners can store a small structured document, a tagline,
category, tags, links and a full About page, inside their subnet's on-chain
identity. tao.app renders it directly from chain, with no off-chain
registration. This document specifies the format so any explorer, wallet or
tool can read and write it.

Everything here is derived from the reference implementations linked at the
end. Where this text and the reference code disagree, the code wins.

## 1. Where it lives

Subtensor's `SubnetIdentitiesV3` storage holds eight `Vec<u8>` fields per
subnet, set by the owner coldkey through
`subtensorModule.setSubnetIdentity`. The payload lives in the `additional`
field, which is limited by the runtime to **1024 bytes**. Every other field
keeps its normal meaning: `description` stays plain prose for wallets and
explorers that know nothing about this format.

## 2. Wire format

```
additional = "TAOAppv1:" + base64url_nopad( deflate_raw( json, level=9, dictionary=DICTIONARY_V1 ) )
```

| Part          | Detail                                                                                                                           |
|---------------|----------------------------------------------------------------------------------------------------------------------------------|
| Prefix        | The 9 ASCII bytes `TAOAppv1:`. The digit is the major version.                                                                   |
| Compression   | Raw DEFLATE (RFC 1951), no zlib or gzip header, level 9, with a preset dictionary.                                               |
| Dictionary    | A frozen 26,414-byte file, SHA-256 `b6b934272c4d315ec32051797f9b024d34e33fa0570cccfafd8cc68acc1d8c6d`. It is part of the format. |
| Text encoding | Base64url (RFC 4648 §5, alphabet `A-Z a-z 0-9 - _`) with no `=` padding.                                                         |
| Budget        | 1024 − 9 = 1015 base64 characters, which is 761 compressed bytes.                                                                |

Why each choice:

- **Text, not raw bytes.** Some pipelines decode `Vec<u8>` as UTF-8 when it
  happens to be valid and as hex otherwise. A base64url string survives every
  one of them unchanged. Raw bytes would gain 33% capacity and lose
  portability.
- **A shared dictionary.** On the 11 curated About pages tao.app had when v1
  was designed, plain DEFLATE fit 6 of them in the budget; DEFLATE with the
  dictionary fit 10. A full About page with four paragraphs, a roadmap and
  links typically encodes to 350–450 bytes.
- **Frozen.** Changing one byte of the dictionary makes every payload already
  on chain undecodable. A new dictionary is a new major version with a new
  prefix. v1 readers ignore other majors.

### 2.1 Plain form

For hand authoring without compression, this is also a valid v1 value:

```json
{
  "apiVersion": "TAOAppv1",
  "tagline": "Hello subnet"
}
```

Readers accept either form. The `apiVersion` key is removed before the
document is interpreted. This form has no size advantage and is offered only
for convenience.

## 3. Detecting a payload

An `additional` value is a TAOApp payload if, after trimming whitespace, it
matches `^TAOAppv(\d+):` or starts with `{` and contains
`"apiVersion": "TAOAppv<N>"`. The captured number is the major version. A
reader for v1 must ignore any other major and treat the field as opaque text.

Values that do not match are ordinary owner prose and must be shown as such.

## 4. The document

The decompressed bytes are a UTF-8 JSON object. Every key is optional.
Unknown keys must be ignored so that v1 readers survive additive changes.
Readers should be lenient at the field level: drop a single invalid value
and keep the rest, never reject the whole document over one bad link.

| Key        | Type     | Limit        | Notes                                                                                                           |
|------------|----------|--------------|-----------------------------------------------------------------------------------------------------------------|
| `tagline`  | string   | 140 chars    | One line under the subnet name.                                                                                 |
| `category` | string   | 32 chars     | Free text. Editors should offer presets (`inference`, `training`, `compute`, `data`, `storage`, `agents`, `finance`, `media`, `science`, `other`); readers group case-insensitively. |
| `tags`     | string[] | 8 × 24 chars |                                                                                                                 |
| `links`    | object   |              | Keys from a fixed set, values are `https://` URLs ≤ 200 chars.                                                  |
| `about`    | object   |              | The About page, see below.                                                                                      |

Link keys: `x`, `telegram`, `docs`, `dashboard`, `whitepaper`, `huggingface`,
`youtube`, `linkedin`, `github`.

### 4.1 `about`

| Key                                           | Type     | Limit                                                                                                               |
|-----------------------------------------------|----------|---------------------------------------------------------------------------------------------------------------------|
| `title`                                       | string   | 120 chars                                                                                                           |
| `subtitle`                                    | string   | 200 chars                                                                                                           |
| `problem`, `solution`, `validators`, `miners` | string   | 3000 chars each                                                                                                     |
| `future`                                      | string[] | 12 × 300 chars. A leading `Title: ` in an item is rendered as a bold label.                                         |
| `benchmark`                                   | object   | `benchmark_description` ≤ 600, `benchmark_url` https ≤ 300, `benchmark_button_text` ≤ 60, `benchmark_subtext` ≤ 200 |
| `team`                                        | object[] | ≤ 10 of `{ name ≤ 80 (required), description ≤ 200, twitter https ≤ 200, linkedin https ≤ 200 }`                    |
| `read_more`                                   | object[] | ≤ 10 of `{ title ≤ 80 (required), subtitle ≤ 200, url https ≤ 300 (required) }`                                     |

Limits are in Unicode characters for text and are enforced by writers. A
reader that finds a longer value should truncate or drop it, not fail.

### 4.2 Rendering rules

- Every string is untrusted text. Render it as text, never as HTML or
  Markdown. Do not interpret URLs found inside prose.
- All URLs in `links`, `benchmark_url`, `team[].twitter`, `team[].linkedin`
  and `read_more[].url` must use the `https:` scheme. Reject anything else.
- The payload is authorised only by the owner coldkey's signature on
  `setSubnetIdentity`. Treat it with the same trust as any other identity
  field the owner controls, which is to say none beyond that.

## 5. Canonical JSON

Writers should emit canonical JSON so that identical content encodes to
identical bytes and diffs against chain state are meaningful:

- No whitespace.
- Keys in the order listed in section 4, nested objects likewise.
- Empty strings, empty arrays and empty objects omitted.

Canonical form matters for comparison, not for validity. A reader must
accept any key order.

### 5.1 Two encoders, two byte strings

DEFLATE output is not unique. Given the same input and dictionary, JavaScript's
pako and Python's zlib produce different compressed bytes that decode to the
same JSON. From `test-vectors.json`:

```
json : {"tagline":"Decentralized inference for everyone","category":"inference","tags":["llm","inference","api"],"links":{"x":"https://x.com/example","telegram":"https://t.me/example","docs":"https://docs.example.ai/","github":"https://github.com/example/subnet"}}
pako : TAOAppv1:Q07XLuhnQEPrSvBoDagHmp-XSjjt5-TkKumgyCQWZBLIDakViaCyHU-mQKhAyxvgEWaoLHh7EP4ECFUJHTJQqq0FAA
zlib : TAOAppv1:Q07XLuhnQEPrSvBoDagHCuxiEE77OTm5QB6yDLC3SyA3pFYkgsp2PJkCoQItb4BHmKGy4O1B-BMgVCV0yECpthYA
```

Therefore: **compare payloads by decoded content, never by the encoded
string.** A tool that re-encodes an unchanged document and submits the new
bytes wastes a transaction and may even fail the size limit where the
original fit. tao.app's own tool submits the chain's existing bytes when the
decoded content is unchanged.

## 6. Encoding and decoding

Both samples below use `dictionary.v1.txt` from this folder. Verify its
SHA-256 against `SHA256SUMS` before use.

### 6.1 JavaScript (Node or browser, pako ≥ 2)

```js
import * as pako from 'pako'; // pako 3 has no default export; `const pako = require('pako')` in CommonJS

const PREFIX = 'TAOAppv1:';
const dictionary = new TextEncoder().encode(DICTIONARY_V1); // contents of dictionary.v1.txt, as a string

const B64 = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';

function base64UrlEncode(bytes) {
    let out = '';
    let i = 0;
    for (; i + 2 < bytes.length; i += 3) {
        const n = (bytes[i] << 16) | (bytes[i + 1] << 8) | bytes[i + 2];
        out += B64[n >> 18] + B64[(n >> 12) & 63] + B64[(n >> 6) & 63] + B64[n & 63];
    }
    if (i < bytes.length) {
        const n = (bytes[i] << 16) | ((i + 1 < bytes.length ? bytes[i + 1] : 0) << 8);
        out += B64[n >> 18] + B64[(n >> 12) & 63];
        if (i + 1 < bytes.length) out += B64[(n >> 6) & 63];
    }
    return out;
}

function base64UrlDecode(text) {
    const clean = text.replace(/=+$/, '');
    if (clean.length % 4 === 1) return null;
    const out = new Uint8Array(Math.floor((clean.length * 3) / 4));
    let buffer = 0, bits = 0, index = 0;
    for (const ch of clean) {
        const value = B64.indexOf(ch === '+' ? '-' : ch === '/' ? '_' : ch);
        if (value < 0) return null;
        buffer = (buffer << 6) | value;
        bits += 6;
        if (bits >= 8) {
            bits -= 8;
            out[index++] = (buffer >> bits) & 0xff;
        }
    }
    return out;
}

/** payload: a plain object following section 4. Returns the `additional` string. */
export function encodePayload(payload) {
    const json = new TextEncoder().encode(JSON.stringify(payload)); // canonicalise first in production
    const compressed = pako.deflateRaw(json, {level: 9, dictionary});
    const additional = PREFIX + base64UrlEncode(compressed);
    if (new TextEncoder().encode(additional).length > 1024) {
        throw new Error('payload does not fit in 1024 bytes');
    }
    return additional;
}

/** Returns the decoded object, or null for anything that is not a v1 payload. */
export function decodePayload(additional) {
    if (!additional) return null;
    const text = additional.trim();
    if (text.startsWith('{')) {
        try {
            const doc = JSON.parse(text);
            if (doc && doc.apiVersion === 'TAOAppv1') {
                const {apiVersion, ...rest} = doc;
                return rest;
            }
        } catch {
        }
        return null;
    }
    if (!text.startsWith(PREFIX)) return null;
    const bytes = base64UrlDecode(text.slice(PREFIX.length));
    if (!bytes || bytes.length === 0) return null;
    try {
        const json = new TextDecoder('utf-8', {fatal: true}).decode(
            pako.inflateRaw(bytes, {dictionary})
        );
        const doc = JSON.parse(json);
        return doc && typeof doc === 'object' && !Array.isArray(doc) ? doc : null;
    } catch {
        return null;
    }
}
```

`Buffer.from(bytes).toString('base64url')` and `Buffer.from(text, 'base64url')`
are equivalent replacements for the two helpers on Node ≥ 16.

### 6.2 Python (standard library only)

```python
import base64
import json
import zlib
from pathlib import Path

PREFIX = "TAOAppv1:"
DICTIONARY = Path("dictionary.v1.txt").read_bytes()  # the 26,414-byte file


def encode_payload(payload: dict) -> str:
    """payload: a dict following section 4. Returns the `additional` string."""
    raw = json.dumps(payload, separators=(",", ":"), ensure_ascii=False).encode("utf-8")
    compressor = zlib.compressobj(9, zlib.DEFLATED, -zlib.MAX_WBITS, 9, zlib.Z_DEFAULT_STRATEGY, zdict=DICTIONARY)
    compressed = compressor.compress(raw) + compressor.flush()
    additional = PREFIX + base64.urlsafe_b64encode(compressed).decode("ascii").rstrip("=")
    if len(additional.encode("utf-8")) > 1024:
        raise ValueError("payload does not fit in 1024 bytes")
    return additional


def decode_payload(additional: str | None) -> dict | None:
    """Returns the decoded dict, or None for anything that is not a v1 payload."""
    if not additional:
        return None
    text = additional.strip()
    if text.startswith("{"):
        try:
            doc = json.loads(text)
        except ValueError:
            return None
        if isinstance(doc, dict) and doc.get("apiVersion") == "TAOAppv1":
            return {k: v for k, v in doc.items() if k != "apiVersion"}
        return None
    if not text.startswith(PREFIX):
        return None
    body = text[len(PREFIX):]
    try:
        raw = base64.urlsafe_b64decode(body + "=" * (-len(body) % 4))
        decompressor = zlib.decompressobj(-zlib.MAX_WBITS, zdict=DICTIONARY)
        data = decompressor.decompress(raw) + decompressor.flush()
        doc = json.loads(data.decode("utf-8"))
    except (ValueError, zlib.error, UnicodeDecodeError):
        return None
    return doc if isinstance(doc, dict) else None
```

### 6.3 Worked example

```
payload    : {"tagline":"Hello subnet"}
additional : TAOAppv1:Q07XHqk5OfnQFpZSLQA
```

Both implementations above produce exactly this string for this input, and
both decode it back to the same object. Note that agreement is not
guaranteed in general, see section 5.1.

### 6.4 Writing it to chain

```
btcli subnets set-identity --netuid <N> --additional-info 'TAOAppv1:…' \
  --subnet-name '…' --github-repo '…' --subnet-contact '…' --subnet-url '…' \
  --discord '…' --description '…' --logo-url '…'
```

`setSubnetIdentity` replaces the whole struct, so pass every field, including
the ones you are not changing. With polkadot-js, pass each `Bytes` argument as
`compactAddLength(stringToU8a(value))`: a bare `Uint8Array` is read as already
SCALE-encoded, and a string starting with `0x` is read as hex.

## 7. Compatibility rules for readers

1. Trim whitespace before detection; on-chain values often end in a newline.
2. Unknown major version: treat the field as opaque text and do not render it.
3. Decompression or JSON failure: treat as opaque text. Do not cache the
   failure if the cause could be transient (a failed library load).
4. Unknown keys: ignore. Invalid values: drop the value, keep the document.
5. Never display the encoded string itself to end users.

## 8. Files and test vectors

This specification ships as a folder of four files:

| File | Purpose |
|---|---|
| `SPEC.md` | This document. |
| `dictionary.v1.txt` | The frozen DEFLATE preset dictionary, 26,414 bytes. Part of the wire format. |
| `test-vectors.json` | Four documents, each encoded once by pako (`encoded_ts`) and once by zlib (`encoded_py`). |
| `SHA256SUMS` | Checksums of the two data files. |

A conforming implementation must decode all eight encoded strings in
`test-vectors.json` to the listed documents. Its own encoder need not
reproduce either byte string; see section 5.1.

Verify the dictionary before trusting it:

```
sha256sum -c SHA256SUMS
```

The expected dictionary hash is
`b6b934272c4d315ec32051797f9b024d34e33fa0570cccfafd8cc68acc1d8c6d`, and the
`dictionary_sha256` field inside `test-vectors.json` carries the same value.

### 8.1 Reading payloads without decoding

tao.app serves the decoded document for any subnet:

```
GET https://api.tao.app/api/beta/subnets/owner-content?netuid=<N>
```

It returns `{ netuid, api_version, encoded_bytes, payload }`, or 404 when the
subnet has no TAOApp payload. Treat every value in `payload` as untrusted
text, exactly as if you had decoded it yourself.
