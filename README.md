# BEACON Protocol
### Blockchain-Encrypted Algorand Connection Over Note-field

**Status:** RFC — Request for Comments  
**Version:** 0.4  
**Author:** [@loafpickleWW](https://github.com/loafpickleWW)  
**Discussion:** Join the Discussions tab or open an issue on this repo

---

## Summary

BEACON is a serverless, privacy-preserving signaling protocol that uses the Algorand note field to bootstrap peer-to-peer WebRTC connections between wallets — with no brokers, no relays, no third-party infrastructure, and no on-chain relationship metadata between communicating parties.

A wallet opts into BEACON with a single one-time key announcement transaction. From that point, any other BEACON participant can send them an encrypted session offer knowing only their Algorand address. The recipient's wallet is the only thing that can derive the decryption key — no private key is ever exposed to any application.

---

## Motivation

Establishing a direct encrypted connection between two parties today requires a signaling server — a broker that helps both sides exchange WebRTC session descriptions before the peer-to-peer channel opens. This is a meaningful point of centralisation: the broker knows who is connecting to whom, when, and from where. 

Furthermore, this reliance creates 3rd-party vendor lock-in. Traditional infrastructure requires someone to pay for hosting, maintain servers, and ensure uptime. If the company hosting the signaling server shuts down or changes their terms, your application breaks.

Algorand's note field is an underutilised primitive. At 1KB, it is large enough to carry an encrypted connection offer. At 0.001 ALGO per transaction, it is cheap enough to use as ephemeral signaling infrastructure. By replacing proprietary servers with a public, decentralized blockchain, we can build open protocols that never go down, cannot be censored, and belong to no single vendor.

BEACON replaces the signaling broker entirely.

---

## Core Design

### The Protocol Address

BEACON uses an open, shared protocol address as a public noticeboard. The protocol is completely open, so anyone can configure their client to monitor any address they choose. All BEACON traffic for a shared instance — announcements, offers, answers, pings, rejections — is sent to this address as 0 ALGO transactions with note fields.

**How Crypto Solves Vendor Lock-in:** 
By using a shared protocol address on a public ledger, BEACON removes the need for hosted backends. The blockchain acts as an immutable, globally available message bus. Developers don't have to worry about API rate limits from private companies, recurring server costs, or infrastructure deprecation. The network provides the service, paid for per-use (fractions of a cent) directly by the users, ensuring the signaling layer will outlive any individual application or company.

Every BEACON client polls this address for new transactions. For encrypted message types, decryption is the inbox filter: if a note decrypts successfully with your derived web key, it is addressed to you. If it does not, it is silently ignored.

**Privacy properties of this design:**

- The sender's address is visible on-chain. The recipient's address never appears in offer or answer transactions.
- Offer and answer note content is encrypted — an observer learns nothing about intent, recipient, or outcome.
- All clients poll the same address, so polling behaviour reveals nothing about who you are expecting to hear from.
- There is no on-chain link between any two communicating wallets in the signaling phase.
- The WebRTC session token is never present in any note — it is derived from the shared secret entirely out of band.

---

## Encryption Model

### The Fundamental Constraint

Web wallets (Pera, Lute, Defly, etc.) intentionally never expose a wallet's raw private key to any application. This is correct security behaviour. BEACON's encryption model is designed around this constraint — no mnemonic, no raw key access, ever.

Algorand's native keypair (ed25519) cannot be used directly for NaCl box encryption (curve25519), and the private key is inaccessible anyway. BEACON solves this with a **signature-derived web key**: a deterministic curve25519 keypair derived by asking the wallet to sign a fixed domain message. The signature is used as entropy — same wallet, same message, same keypair every time.

### Web Key Derivation

The domain transaction is constructed with **fully fixed parameters** — no live network fields. This is the critical design requirement: because ed25519 signing is deterministic by specification (same key + same message = same signature, always), freezing every transaction field guarantees the identical signature is produced on any device, any browser, any session, forever.

The transaction is intentionally unbroadcastable — `firstValid` and `lastValid` are zeroed, meaning it could never be submitted to the network. This is a security feature: it provably cannot leak onto the chain, and the domain note string `BEACON/1:derive-encryption-key` never appears in any public record.

**Canonical fixed parameters — all implementations must use these exactly:**

```javascript
const BEACON_GENESIS_HASH_MAINNET = "wGHE2Pwdvd7S12BL5FaOP20EGYesN73ktiC1qzkkit8=";
const BEACON_GENESIS_HASH_TESTNET = "SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=";

function buildDomainTransaction(activeAddress, genesisHash) {
  return algosdk.makePaymentTxnWithSuggestedParamsFromObject({
    from: activeAddress,
    to: activeAddress,
    amount: 0,
    note: new TextEncoder().encode("BEACON/1:derive-encryption-key"),
    suggestedParams: {
      genesisHash,                // hardcoded — never changes for a given network
      genesisID: "mainnet-v1.0", // hardcoded
      firstValid: 0,             // zeroed — transaction is unbroadcastable by design
      lastValid: 0,              // zeroed — transaction is unbroadcastable by design
      fee: 0,
      flatFee: true,
      minFee: 1000,
    }
  });
}

// Derive the web keypair
const domainTxn = buildDomainTransaction(activeAddress, BEACON_GENESIS_HASH_MAINNET);
const signed = await signTransactions([algosdk.encodeUnsignedTransaction(domainTxn)]);
const sigBytes = algosdk.decodeSignedTransaction(signed[0]).sig;

// Derive deterministic curve25519 secret key from signature
const webSecretKey = blake2b(sigBytes, { dkLen: 32 });
const webKeypair = nacl.box.keyPair.fromSecretKey(webSecretKey);
// webKeypair.publicKey  → the Web Public Key (wpk) to announce
// webKeypair.secretKey  → used locally for decryption, never leaves the client
```

The wallet prompt occurs once per session. Because all parameters are fixed and ed25519 is deterministic, the same wallet always produces the identical signature, recovering the identical curve25519 keypair on any device. No key material is stored server-side or in any database.

**Implementations must not substitute live `suggestedParams` from the network.** Fields like `firstValid`, `lastValid`, and `fee` change constantly — any variation produces a different signature and therefore a different keypair, permanently breaking decryption of previously received messages.

### Key Announcement

Before a wallet can receive BEACON messages, it performs a **one-time key announcement**: a single transaction to the BEACON protocol address publishing its Web Public Key in plaintext. This is the only public information BEACON requires.

Senders query the indexer for a recipient's most recent `announce` transaction before encrypting. The announced `wpk` is the encryption target. Decryption requires re-deriving the web keypair via the same signature process — which only the key owner can do.

```
Sender knows:     recipient's Algorand address
Sender queries:   BEACON address for announce txn from that address  
Sender uses:      announced wpk to encrypt the offer
Receiver uses:    wallet signature → derived secret key → decrypts offer
```

This is the only keypair system in play. There is no cross-algorithm conversion, no registry beyond the on-chain announcements, and no private key exposure.

---

## Message Types

| Type | Encrypted | Description |
|---|---|---|
| `announce` | No | One-time Web Public Key publication. Plaintext — intended to be publicly readable. |
| `offer` | Yes | Initiator signals session availability. Encrypted to recipient's `wpk`. |
| `answer` | Yes | Responder confirms readiness. Encrypted to initiator's `wpk`. |
| `reject` | Yes | Explicit decline. Encrypted so only the initiator sees it. |
| `ping` | Yes | Presence beacon, no session implied. |
| `revoke` | Yes | Cancel a pending offer before expiry. |
| `announce-rotate` | No | Supersedes a previous announcement with a new `wpk`. See Key Rotation. |

---

## Note Structure

All notes are prefixed with `BEACON/1:` for indexer filtering, followed by the message payload.

**Announce (plaintext):**
```
BEACON/1:<base64(JSON)>
```
```json
{
  "proto": "BEACON/1",
  "type": "announce",
  "wpk": "<curve25519 web public key, base64>",
  "ts": 1718000000
}
```

**Encrypted types (offer, answer, reject, ping, revoke):**
```
BEACON/1:<base64(nacl.box(JSON, nonce, recipientWpk, ephemeralSecretKey))>
```

The outer wrapper includes the ephemeral public key and nonce needed for decryption:
```json
{
  "epk": "<ephemeral curve25519 pubkey, base64>",
  "nonce": "<base64>",
  "ct": "<ciphertext base64>"
}
```

The decrypted inner payload for `offer` and `answer`:
```json
{
  "proto": "BEACON/1",
  "type": "offer",
  "wpk": "<sender's web public key, base64>",
  "ts": 1718000000,
  "exp": 1718003600
}
```

| Field | Description |
|---|---|
| `proto` | Protocol version string |
| `type` | Message type |
| `wpk` | Sender's web public key — recipient uses this to derive the shared secret and compute the session token |
| `ts` | Unix timestamp of issue — used in session token derivation |
| `exp` | Unix timestamp of expiry — clients must ignore stale messages |

Note the absence of a `pid` field. The WebRTC session token is derived, never transmitted.

---

## Session Token Derivation

The WebRTC peer ID never appears in any note payload. Both sides independently derive a one-time session token from the NaCl shared secret:

```javascript
const sharedSecret = nacl.box.before(senderWpk, myWebSecretKey);
const sessionToken = blake2bHex(
  concatBytes(sharedSecret, toBytes(offer.ts))
).slice(0, 32);
```

Both sides compute the same token without communicating it. The initiator registers as a WebRTC peer with `id = sessionToken`. The responder derives the same token and connects to it. The peer ID never touches the chain and never appears in any log in a form linkable to either wallet.

---

## Full Session Flow

```
                        One-Time Setup (Receiver — done once, ever)
                        ─────────────────────────────────────────
Receiver                         Algorand (BEACON Address)
  │                                        │
  │  Sign domain message                   │
  │  Derive curve25519 web keypair         │
  │  Broadcast announce with wpk ─────────▶│
  │                                        │
  │  Setup complete. wpk is now publicly   │
  │  queryable by anyone who knows         │
  │  receiver's Algorand address.          │


                        Session Establishment
                        ─────────────────────────────────────────
Sender                           Algorand (BEACON Address)            Receiver
  │                                        │                               │
  │  1. Query BEACON address for           │                               │
  │     announce from receiver's address   │                               │
  │     → retrieve receiver's wpk         │                               │
  │                                        │                               │
  │  2. Sign domain message                │                               │
  │     Derive own web keypair             │                               │
  │     Generate ephemeral keypair         │                               │
  │     Encrypt offer to receiver's wpk   │                               │
  │     Offer contains: own wpk + ts/exp  │                               │
  │                                        │                               │
  │──── 0 ALGO + BEACON offer note ───────▶│                               │
  │                                        │                               │
  │                                        │◀──── polling every N secs ────│
  │                                        │      sign domain message      │
  │                                        │      re-derive web keypair    │
  │                                        │      try decrypt each note    │
  │                                        │                               │
  │                                        │   Receiver's key decrypts ✓   │
  │                                        │   Extract sender's wpk        │
  │                                        │   Derive sessionToken         │
  │                                        │   from sharedSecret + ts      │
  │                                        │                               │
  │                                        │   Encrypt answer to           │
  │                                        │   sender's wpk                │
  │                                        │                               │
  │◀──── polling every N secs ─────────────│◀─── 0 ALGO + BEACON answer ───│
  │  Sender's key decrypts ✓               │                               │
  │  Derive same sessionToken              │                               │
  │  from sharedSecret + ts               │                               │
  │                                        │                               │
  │  Register WebRTC peer                  │      Connect to               │
  │  as sessionToken                       │      sessionToken peer ID     │
  │                                        │                               │
  │◀═══════════════════ direct WebRTC P2P connection ════════════════════▶│
  │                                        │                               │
  │              chain no longer involved after this point                 │
```

---

## Key Rotation

A wallet may rotate its web key at any time by broadcasting an `announce-rotate` transaction:

```json
{
  "proto": "BEACON/1",
  "type": "announce-rotate",
  "wpk": "<new curve25519 web public key, base64>",
  "supersedes": "<transaction ID of previous announce or announce-rotate>",
  "ts": 1718000000
}
```

**Sender behaviour:** Always use the most recent `announce` or `announce-rotate` from the target address, ordered by round. If a chain of `supersedes` references is present, follow it to the tip.

**Rotation scenarios:**
- Device compromise → rotate immediately, all future offers encrypt to new key
- Routine key hygiene → rotate periodically, old offers using the previous key become undeliverable
- Lost key recovery → re-derive from wallet signature using a new domain string (versioned: `BEACON/1:derive-encryption-key:v2`), announce-rotate with the new key

Past messages encrypted to the old key are permanently unreadable after rotation. This is intentional — rotation provides forward security for future sessions.

---

## Indexer Polling

Clients poll the BEACON protocol address using any public Algorand indexer:

```
GET /v2/accounts/{BEACON_ADDRESS}/transactions
  ?note-prefix=BEACON/1:
  &min-round={lastSeenRound}
  &limit=50
```

`lastSeenRound` is persisted in localStorage and updated after each successful poll. Clients never reprocess old traffic. The fixed address and shared note prefix makes this query highly cache-friendly at the indexer level — every BEACON client in existence makes an identical request.

For key announcement lookup, senders query by sender address:

```
GET /v2/accounts/{BEACON_ADDRESS}/transactions
  ?note-prefix=BEACON/1:
  &sender={recipientAddress}
  &limit=5
```

Filter results for `announce` or `announce-rotate` types and take the most recent.

**Suggested polling intervals:**

| Client state | Interval |
|---|---|
| Active / listening | 5 seconds |
| Idle / background | 30 seconds |
| Offer pending | 5 seconds until answer or expiry |

---

## Security Considerations

**Cross-device determinism**
The web keypair is identical across all devices and browsers for the same wallet because ed25519 signing is deterministic by specification — same private key plus same message always produces the same signature, regardless of platform. The fixed transaction parameters ensure the message is identical everywhere. Implementations must never introduce variable fields (live network params, timestamps, random nonces) into the domain transaction. Any variation silently produces a different keypair and permanently breaks decryption of existing messages with no error visible to the user.

**Domain transaction not broadcastable by design**
The domain transaction has `firstValid: 0` and `lastValid: 0`, making it invalid for network submission. This is intentional. It ensures the domain note string `BEACON/1:derive-encryption-key` never appears on-chain, preventing observers from identifying BEACON participants via their key derivation activity. The only BEACON activity visible on-chain is the `announce` transaction, which is expected to be public.

**Signature-derived key security**
The web keypair is derived from a wallet signature over a fixed domain message. The security assumption is that the ed25519 signature is indistinguishable from random to anyone who does not hold the private key, making the derived curve25519 key computationally infeasible to recover without wallet access. This is the same assumption underlying similar constructions in production protocols.

**Domain separation**
The domain message `BEACON/1:derive-encryption-key` is specific to this protocol and version. Implementations must not reuse signatures from other contexts for key derivation. Future versions must use a distinct domain string.

**Replay attacks**
The `exp` field provides expiry. Clients must reject any message where `exp` has passed or where `ts` is more than 60 seconds in the future (clock skew tolerance).

**Announce freshness**
A stale `announce` transaction cannot be revoked — it remains on-chain. Key rotation via `announce-rotate` is the correct mitigation. Clients should warn users if the most recent announcement is older than a configurable threshold (suggested: 90 days).

**Spam**
The 0.001 ALGO minimum transaction fee provides natural rate limiting. Client-side filtering by sender address drop rate per round is recommended for resilience against targeted spam.

**Sender anonymity**
The sender's Algorand address is visible on-chain in offer transactions. For stronger sender anonymity, clients may use a throwaway session wallet funded with a small amount of ALGO. The protocol does not mandate this but implementations should make it easy.

**Forward secrecy**
Per-session ephemeral keypairs provide partial forward secrecy at the signaling layer — compromise of the web keypair does not expose past session tokens, since each session uses a fresh ephemeral key. Full forward secrecy at the communication layer depends on the WebRTC implementation. Key rotation provides forward secrecy for future sessions.

**Multisig and rekeyed accounts**
Web key derivation requires a single wallet to sign the domain message. Multisig accounts cannot derive a web key without a custom m-of-n signing ceremony. Rekeyed accounts sign with the authorised key rather than the address-derived key — derivation will produce a different web keypair depending on which key signs. This is a known limitation and a candidate for a future revision.

---

## Open Questions for Discussion

1. **Protocol address selection** — Should this be a known burn address, a vanity address, or a multisig with no valid signers? What are the tradeoffs?

2. **Announce expiry** — Should `announce` transactions carry an `exp` field? Forcing rotation adds friction but improves hygiene. Should clients refuse to send to stale announcements?

3. **Session token derivation** — Is BLAKE2b the right choice? Should the derivation include additional entropy beyond shared secret and timestamp?

4. **Note size constraints** — 1KB is sufficient for v1 but limits future extensibility. Should larger payloads reference an IPFS CID?

5. **Forward secrecy** — Is full double-ratchet style forward secrecy a v1 requirement or acceptable to defer?

6. **Multisig support** — Is there a clean way to support multisig accounts in the key derivation step?

7. **Polling vs WebSocket** — AlgoNode and other indexers are exploring WebSocket support. Should BEACON specify a WebSocket mode for lower-latency delivery?

8. **ARC track** — When ready to formalise, which ARC track is most appropriate?

---

## Dependencies

| Library | Purpose |
|---|---|
| `algosdk` | Address decoding, transaction construction, simulate |
| `tweetnacl` | NaCl box encryption, decryption, ephemeral keypair generation |
| `@noble/hashes` | BLAKE2b for web key derivation and session token derivation |

All are small, audited, and have no server-side requirements. The ed25519→curve25519 conversion (`ed2curve`) used in earlier drafts has been removed — it is no longer part of the protocol.

---

## Reference Implementation

BEACON/1 is currently implemented in [Wen Tools P2P Chat](https://wentools.xyz) — a serverless, wallet-authenticated encrypted chat and file transfer tool built on Algorand. The implementation uses `@txnlab/use-wallet-react` for wallet connection, `tweetnacl` for encryption, and the AlgoNode public indexer for polling (no API key required).

---

## How to Give Feedback

This is a draft RFC — nothing here is final. Open an issue on this repo to discuss any aspect of the protocol. Specifically looking for:

- Security review of the signature-derived key construction
- Review of the key announcement and rotation model
- Thoughts on the session token derivation scheme
- Any Algorand-specific edge cases (rekeyed accounts, multisig, ledger hardware wallets)
- Use cases beyond P2P chat this design could serve

Pull requests to improve the spec are welcome.

---

## Changelog

| Version | Changes |
|---|---|
| 0.4 | Replaced generic domain transaction construction with canonical fixed-parameter recipe. Added `BEACON_GENESIS_HASH` constants. Documented unbroadcastable transaction as intentional security feature. Added Cross-device determinism and Domain transaction security considerations. Clarified that live `suggestedParams` must never be used. |
| 0.3 | Replaced native ed25519→curve25519 conversion with signature-derived web key model. Added one-time `announce` message type. Added `announce-rotate` for key rotation with `supersedes` chaining. Removed `ed2curve` dependency. Clarified fundamental private key constraint and why native key conversion is not used. |
| 0.2 | Replaced curve25519 + NFD key registry with native ed25519→curve25519 address derivation. Added simulate-based passive authentication. Removed `pid` from note payload in favour of derived session tokens. |
| 0.1 | Initial draft |

---

## License

This spec is released into the public domain. Implement it wherever you like.
