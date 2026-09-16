# TAOAppv1 subnet identity payload

A compact, versioned format for structured subnet content (tagline, category,
tags, links, About page) stored in the `additional` field of a Bittensor
subnet's on-chain identity. tao.app renders it directly from chain.

- **`SPEC.md`**: the format, the document schema, rendering and compatibility
  rules, and complete encoder/decoder samples in JavaScript and Python.
- **`dictionary.v1.txt`**: the frozen DEFLATE preset dictionary. Required to
  encode or decode. Never modify it; a changed dictionary is a new version.
- **`test-vectors.json`**: eight encoded strings and their decoded documents.
  A conforming reader decodes all eight.
- **`SHA256SUMS`**: checksums for the two data files.

Quick start, Python 3.11+, standard library only:

```python
import base64, json, zlib
from pathlib import Path

DICT = Path("dictionary.v1.txt").read_bytes()

def decode(additional: str) -> dict | None:
    text = additional.strip()
    if not text.startswith("TAOAppv1:"):
        return None
    body = text[len("TAOAppv1:"):]
    raw = base64.urlsafe_b64decode(body + "=" * (-len(body) % 4))
    d = zlib.decompressobj(-zlib.MAX_WBITS, zdict=DICT)
    return json.loads(d.decompress(raw) + d.flush())

print(decode("TAOAppv1:Q07XHqk5OfnQFpZSLQA"))  # {'tagline': 'Hello subnet'}
```

See `SPEC.md` section 6 for the full, hardened implementations.

Status: v1, frozen, published 2026-09-10.
