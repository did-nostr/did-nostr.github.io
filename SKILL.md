# did:nostr - A Nostr Key as a Decentralized Identifier

## Purpose

Turn a Nostr public key into a [W3C DID](https://www.w3.org/TR/did-core/) so any app
can resolve it to a DID document, verify its key, and bridge the identity into WebID,
Solid, Verifiable Credentials, and HTTP auth. There is no registry, no ledger, and
nothing to mint: the keypair the user already has on Nostr **is** the identifier.

```
did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2
```

## Core Idea

- The identifier is the **64-character lowercase hex** secp256k1 public key (BIP-340 x-only).
- `npub...` (bech32) is **display only**. Always convert to hex for the DID.
- A conforming resolver can produce a valid DID document from the key **alone, offline**.
  HTTP and relay tiers only enrich it.

## The DID Format

```
did:nostr:<64-char-lowercase-hex-pubkey>
```

Convert an `npub` to the hex form before building a DID:

```js
import { nip19 } from 'nostr-tools'

const { data: hex } = nip19.decode('npub1cpxejnc58zpcuyh0pt8gvkzpv34qxceu0sqp7jec2nk9nut7p5zs4zyx4c')
const did = `did:nostr:${hex}`
// did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2
```

## The Multikey (Key Encoding)

The key appears in the DID document as a `Multikey` with a `publicKeyMultibase` value.
The encoding is deterministic, so it can be computed with **zero dependencies**:

```
publicKeyMultibase = "f"     base16 multibase prefix
                   + "e701"  secp256k1-pub multicodec (0xe7 as unsigned varint)
                   + "02"    compressed-point parity (Nostr x-only keys imply even Y)
                   + <64-char hex pubkey>
```

```js
// hex pubkey -> publicKeyMultibase
function toMultikey(hexPub) {
  return 'fe70102' + hexPub.toLowerCase()   // f + e701 + 02 + key
}
// fe70102124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2
```

`Multikey` / `publicKeyMultibase` are defined by
[W3C Controlled Identifiers v1.0](https://www.w3.org/TR/cid-1.0/), which the DID document
declares in its `@context`.

## The DID Document

### Minimal (derivable from the key alone, no network)

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://www.w3.org/ns/cid/v1",
    "https://w3id.org/nostr/context"
  ],
  "id": "did:nostr:124c0fa9…bee0fdd2",
  "type": "DIDNostr",
  "verificationMethod": [{
    "id": "did:nostr:124c0fa9…bee0fdd2#key1",
    "type": "Multikey",
    "controller": "did:nostr:124c0fa9…bee0fdd2",
    "publicKeyMultibase": "fe70102124c0fa9…bee0fdd2"
  }],
  "authentication": ["#key1"],
  "assertionMethod": ["#key1"]
}
```

> Note: the `@context` leads with `https://www.w3.org/ns/did/v1` (required by DID Core, spec 0.1.1),
> followed by `https://www.w3.org/ns/cid/v1`, which defines `Multikey` and `publicKeyMultibase`.

### Enhanced (optional fields)

Richer documents may also carry:

- `alsoKnownAs` - links to a [WebID](https://solidproject.org/TR/protocol#webid),
  ActivityPub actor, Bluesky DID, or other identifiers the same key controls.
- `service` - relay endpoints for the key.
- `profile` - the key's `kind 0` metadata (name, picture, ...).
- `follows` - the key's `kind 3` social graph.

## Resolution (Three Tiers)

A resolver **MUST** support tier 1. Tiers 2 and 3 are progressive enhancement.

### Tier 1 - Minimal / offline (no network)

```js
function resolveOffline(did) {
  const pub = did.replace(/^did:nostr:/, '').toLowerCase()
  if (!/^[0-9a-f]{64}$/.test(pub)) throw new Error('invalid did:nostr')
  const id = `did:nostr:${pub}`
  return {
    '@context': ['https://www.w3.org/ns/did/v1', 'https://www.w3.org/ns/cid/v1', 'https://w3id.org/nostr/context'],
    id,
    type: 'DIDNostr',
    verificationMethod: [{
      id: `${id}#key1`,
      type: 'Multikey',
      controller: id,
      publicKeyMultibase: 'fe70102' + pub
    }],
    authentication: ['#key1'],
    assertionMethod: ['#key1']
  }
}
```

### Tier 2 - HTTP (richer document from a pod)

```js
async function resolveHttp(did, domain) {
  const pub = did.replace(/^did:nostr:/, '')
  const res = await fetch(`https://${domain}/.well-known/did/nostr/${pub}.json`)
  if (!res.ok) throw new Error(`resolve failed: ${res.status}`)
  return res.json()   // server-curated DID document
}
```

Hosted resolver/directory: [nostr.social](https://nostr.social/) (built on
[beacon](https://github.com/JavaScriptSolidServer/beacon)).

### Tier 3 - Enhanced (enrich from relays)

```js
// zero-dep relay query
function queryRelay(url, filter, timeout = 4000) {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(url)
    const sub = 'did' + Math.floor(performance.now()).toString(36)
    const t = setTimeout(() => { ws.close(); reject(new Error('timeout')) }, timeout)
    ws.onopen = () => ws.send(JSON.stringify(['REQ', sub, filter]))
    ws.onmessage = (m) => {
      const [type, , ev] = JSON.parse(m.data)
      if (type === 'EVENT') { clearTimeout(t); ws.close(); resolve(ev) }
      if (type === 'EOSE')  { clearTimeout(t); ws.close(); resolve(null) }
    }
    ws.onerror = (e) => { clearTimeout(t); reject(e) }
  })
}

async function fetchProfile(pub, relays = ['wss://relay.damus.io', 'wss://nos.lol']) {
  for (const url of relays) {
    try {
      const ev = await queryRelay(url, { kinds: [0], authors: [pub], limit: 1 })
      if (ev) return JSON.parse(ev.content)   // { name, picture, about, ... }
    } catch { /* try next relay */ }
  }
  return null
}
```

### Compose the tiers

```js
async function resolve(did, { domain, relays } = {}) {
  if (domain) {
    try { return await resolveHttp(did, domain) } catch { /* fall through */ }
  }
  const doc = resolveOffline(did)              // always works
  if (relays) {
    const pub = did.replace(/^did:nostr:/, '')
    const profile = await fetchProfile(pub, relays)
    if (profile) doc.profile = profile
  }
  return doc
}
```

## Authenticating as a WebID (NIP-98 + LWS)

The same key both **resolves** and **authenticates**. Declare the did:nostr key as a
`Multikey` verification method in a WebID profile (and/or `alsoKnownAs` from the DID
document to that WebID). A server that resolves did:nostr → WebID can then treat a
[NIP-98](https://nip98.com/) signed HTTP request as authenticating **that WebID**, not
merely the `did:nostr` identifier. This is the binding standardized by W3C
[Linked Web Storage (LWS) 1.0](https://www.w3.org/TR/2026/WD-lws10-authn-ssi-cid-20260423/),
which is built on CID v1.0.

```
1. User signs an HTTP request with their Nostr key (NIP-98, kind 27235).
2. Server verifies the signature -> recovers the pubkey -> did:nostr:<pubkey>.
3. Server resolves the DID / WebID and confirms the key is a declared verification method.
4. Request is authenticated as the WebID. One keypair, no passwords, no sessions.
```

For the full NIP-98 client + server code, see [nip98.com/SKILL.md](https://nip98.com/SKILL.md).

## Building Apps (Patterns)

- **Sign in with a Nostr key, get a portable identity.** Take the user's NIP-07 pubkey
  (`await window.nostr.getPublicKey()`), form `did:nostr:<hex>`, resolve it. You now have a
  W3C DID and a verification key with no account system.

- **Resolve any did:nostr to a profile.** `resolveOffline` for the verifiable key, then
  Tier 3 (`kind 0`) for display name/avatar. Render trustlessly: the key proves the doc.

- **Bridge a Nostr identity into Solid/WebID.** Publish a DID document (Tier 2) whose
  `alsoKnownAs` points at the user's WebID; have the WebID declare the same key. Now the
  identity works in both the Nostr and Solid/Verifiable-Credentials worlds.

- **Authenticate HTTP requests.** Use NIP-98 with the same key (see LWS section). No second
  credential, no session store.

## Key Design Points

- **Self-certifying** - the identifier is the key, so the minimal document needs no network
  and cannot be forged; verification is local.
- **No update / no deactivate** - there is no mutable on-chain state; the DID is immutable.
  Richer (Tier 2/3) data is curated out-of-band and is not part of the core identifier.
- **hex is canonical** - always resolve/compare on the 64-char hex pubkey; treat `npub` as a
  display encoding only.
- **Public by default** - a Nostr pubkey is public and correlatable. Never put secrets in a
  DID document; treat resolution as public information.
- **One keypair, many roles** - the same key resolves, signs Nostr events, and authenticates
  HTTP (NIP-98 / LWS).

## References

- [did:nostr method draft](https://nostrcg.github.io/did-nostr/) - the specification (W3C Nostr Community Group, unofficial draft)
- [did-nostr.com](https://did-nostr.com/) - hub + ecosystem directory
- [W3C DID Core](https://www.w3.org/TR/did-core/) - the DID data model
- [W3C Controlled Identifiers v1.0](https://www.w3.org/TR/cid-1.0/) - defines `Multikey` / `publicKeyMultibase`
- [W3C Linked Web Storage 1.0 (via CID)](https://www.w3.org/TR/2026/WD-lws10-authn-ssi-cid-20260423/) - authenticate a did:nostr key as a WebID
- [NIP-98](https://nip98.com/) - HTTP auth with a Nostr key (`nip98.com/SKILL.md` for code)
- [nostr-did-resolver](https://github.com/block-core/nostr-did-resolver) - resolver library
- [JavaScriptSolidServer](https://github.com/JavaScriptSolidServer/JavaScriptSolidServer) - serves DID docs + did:nostr → WebID auth
- [nostr.social](https://nostr.social/) - live directory + resolver
- GitHub topic [`did-nostr`](https://github.com/topics/did-nostr) - the broader ecosystem
