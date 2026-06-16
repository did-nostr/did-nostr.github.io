# did-nostr.github.io

Source for **[did-nostr.com](https://did-nostr.com)** — an independent hub for the
**did:nostr** DID method (a Nostr public key as a W3C decentralized identifier).

Single static page, no build step. Served via GitHub Pages from the **`gh-pages`** branch.

## What's here

- `index.html` — the site (explainer · resolution steps · DID-document shape · ecosystem directory · resolver stub)
- `SKILL.md` — machine-readable build guide for LLMs and agents (also advertised from the page via `<link rel="alternate" type="text/markdown">`)
- `CNAME` — custom domain (`did-nostr.com`)

## Relationship to the spec

This is the friendly front door, **not** the specification. The did:nostr method draft lives in
the Nostr Community Group at [nostrcg/did-nostr](https://nostrcg.github.io/did-nostr/). This site
is independent and does **not** imply Community-Group consensus.

Sibling site: [nip98.com](https://nip98.com/) (NIP-98 HTTP auth).

## For agents and LLMs

[`SKILL.md`](https://did-nostr.com/SKILL.md) is a code-forward, zero-dependency guide to
building apps with did:nostr: key encoding, the three resolution tiers, the DID-document
shape, and the NIP-98 + LWS "authenticate as a WebID" bridge. The page advertises it via
`<link rel="alternate" type="text/markdown">` so crawlers and agents can auto-discover it.

## Contributing

The ecosystem directory is the living part — open a PR against `index.html` to add a resolver,
bridge, library, or client.

## Deploy

GitHub Pages → Settings → Pages → Source: **Deploy from a branch** → `gh-pages` / `/ (root)`.
DNS for the apex `did-nostr.com` points A/AAAA records at GitHub Pages.
