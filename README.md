# BEACON Protocol
### Blockchain-Encrypted Algorand Connection Over Note-field

**Status:** RFC — Request for Comments  
**Version:** 0.1  
**Author:** [@loafpickleWW](https://github.com/loafpickleWW)  
**Discussion:** Join the Discussions tab or open an issue on this repo

---

## Summary

BEACON is a serverless, privacy-preserving signaling protocol that uses the Algorand note field to bootstrap peer-to-peer WebRTC connections between wallets — with no brokers, no relays, no third-party infrastructure, and no on-chain relationship metadata between communicating parties.

---

## Motivation

Establishing a direct encrypted connection between two parties today requires a signaling server — a broker that helps both sides exchange WebRTC session descriptions before the peer-to-peer channel opens. This is a meaningful point of centralisation: the broker knows who is connecting to whom, when, and from where. 

Furthermore, this reliance creates 3rd-party vendor lock-in. Traditional infrastructure requires someone to pay for hosting, maintain servers, and ensure uptime. If the company hosting the signaling server shuts down or changes their terms, your application breaks.

Algorand's note field is an underutilised primitive. At 1KB, it is large enough to carry an encrypted connection offer. At 0.001 ALGO per transaction, it is cheap enough to use as ephemeral signaling infrastructure. By replacing proprietary servers with a public, decentralized blockchain, we can build open protocols that never go down, cannot be censored, and belong to no single vendor.

BEACON replaces the signaling broker entirely.

---

## Core Design

### The Protocol Address: `[PROTOCOL_ADDRESS]`

BEACON uses an open, shared protocol address as a public noticeboard. The protocol is completely open, so anyone can configure their client to monitor any address they choose. All BEACON traffic for a shared instance — offers, answers, pings, rejections — is sent to this address as 0 ALGO transactions with encrypted note fields.

**How Crypto Solves Vendor Lock-in:** 
By using a shared protocol address on a public ledger, BEACON removes the need for hosted backends. The blockchain acts as an immutable, globally available message bus. Developers don't have to worry about API rate limits from private companies, recurring server costs, or infrastructure deprecation. The network provides the service, paid for per-use (fractions of a cent) directly by the users, ensuring the signaling layer will outlive any individual application or company.

Every BEACON client polls this address for new transactions. Decryption is the inbox filter: if a note decrypts successfully with your private key, it is addressed to you. If it does not, it is silently ignored.

**Privacy properties of this design:**

- The sender's address is visible on-chain. The recipient's address never appears.
- Note content is encrypted — an observer learns nothing about intent, recipient, or outcome.
- All clients poll the same address, so polling behaviour reveals nothing about who you are expecting to hear from.
- There is no on-chain link between any two communicating wallets.

### Encryption

Notes are encrypted using NaCl box (X25519-XSalsa20-Poly1305):

- Sender generates an ephemeral keypair per message
- Note is encrypted to recipient's known curve25519 public key
- Ephemeral public key is included in the note payload for decryption
- Recipient's address never appears anywhere in the transaction

Public key discovery uses NFD metadata as the primary registry. A wallet publishes their curve25519 public key in their NFD properties. Senders resolve the recipient's NFD to retrieve it. This means any two NFD holders can BEACON each other with zero prior key exchange.

### Note Structure

All notes are prefixed with `BEACON/1:` for indexer filtering, followed by a base64-encoded NaCl box:

```
BEACON/1:<base64(nacl.box(JSON, nonce, recipientPubKey, ephemeralSecretKey))>
```

The decrypted JSON payload:

```json
{
  "proto": "BEACON/1",
  "type": "offer",
  "pid": "<WebRTC peer ID or session token>",
  "pk": "<sender curve25519 pubkey>",
  "ts": 1718000000,
  "exp": 1718003600
}
```

| Field | Description |
|---|---|
| `proto` | Protocol version string |
| `type` | Message type (see below) |
| `pid` | WebRTC peer ID or session identifier |
| `pk` | Sender's curve25519 public key for reply encryption |
| `ts` | Unix timestamp of issue |
| `exp` | Unix timestamp of expiry — clients ignore stale messages |

### Message Types

| Type | Direction | Description |
|---|---|---|
| `offer` | A → noticeboard | Initiator announces session availability |
| `answer` | B → noticeboard | Responder confirms and provides their peer ID |
| `reject` | B → noticeboard | Explicit decline, encrypted so only initiator sees it |
| `ping` | Any → noticeboard | Presence beacon, no session implied |
| `revoke` | A → noticeboard | Cancel a pending offer before it expires |

---

## Full Session Flow

```
Alice                    Algorand (BEACON Address)              Bob
  │                                │                              │
  │  1. Resolve Bob's pubkey       │                              │
  │     via NFD metadata           │                              │
  │                                │                              │
  │  2. Encrypt offer to Bob's     │                              │
  │     pubkey, include own pid    │                              │
  │     and pubkey for reply       │                              │
  │                                │                              │
  │──── 0 ALGO + BEACON note ─────▶│                              │
  │                                │                              │
  │                                │◀──── polling every N secs ───│
  │                                │      try decrypt all notes   │
  │                                │      since last seen round   │
  │                                │                              │
  │                                │    Bob's key decrypts ✓      │
  │                                │    extract Alice's pid       │
  │                                │    and pubkey                │
  │                                │                              │
  │                                │    Bob encrypts answer       │
  │                                │    to Alice's pubkey         │
  │                                │                              │
  │◀──── polling every N secs ─────│◀─── 0 ALGO + BEACON note ───│
  │  Alice's key decrypts ✓        │                              │
  │  extract Bob's pid             │                              │
  │                                │                              │
  │◀═══════════════ direct WebRTC P2P connection ════════════════▶│
  │                                │                              │
  │           chain no longer involved after this point           │
```

---

## Indexer Polling

Clients poll the BEACON protocol address using any public Algorand indexer:

```
GET /v2/accounts/{BEACON_ADDRESS}/transactions
  ?note-prefix=BEACON/1:
  &min-round={lastSeenRound}
  &limit=50
```

`lastSeenRound` is persisted locally and updated after each poll. This ensures clients only process new transactions and never re-process stale traffic. The fixed address and shared note prefix makes this query extremely cache-friendly at the indexer level.

**Suggested polling intervals:**

| Client state | Interval |
|---|---|
| Active / listening | 5 seconds |
| Idle / background | 30 seconds |
| Offer pending | 5 seconds until answer or expiry |

---

## Security Considerations

**Replay attacks**
The `exp` field provides expiry. Clients must reject any message where `ts` is in the future by more than a clock-skew tolerance (suggested: 60 seconds) or where `exp` has passed.

**Spam**
The 0.001 ALGO minimum transaction fee provides natural rate limiting. A dedicated spam filter on the client — tracking sender addresses and drop-rate per round — is recommended for high-traffic deployments.

**Sender identity**
The sender's Algorand address is visible on-chain. For stronger sender anonymity, clients may use a throwaway session wallet funded with a small amount of ALGO and discarded after the session. The protocol does not mandate this but implementations should make it easy.

**Key compromise**
If a curve25519 private key is compromised, past messages encrypted to that key are exposed. Forward secrecy is not provided at the protocol level — this is a known limitation of the v1 spec and a candidate for a future revision.

**Protocol address**
The protocol address is unowned and unspendable. Transactions sent to it are final. This is intentional — no party can censor the noticeboard or selectively block traffic.

---

## Open Questions for Discussion (We Need Your Input!)

This is the most critical section of the RFC. The Algorand developer community thrives on collaboration, and your input here will shape the protocol. Please drop into the **Discussions** tab to share your thoughts on the following:

1. **Protocol address selection** — Should this be a known burn address, a vanity address, or a multisig with no valid signers? What are the tradeoffs?
2. **Public key registry** — NFD is the suggested primary registry but not everyone has an NFD. What should the fallback be? A prior BEACON self-transaction publishing your pubkey? A separate on-chain registry?
3. **Note size constraints** — 1KB is sufficient for v1 but limits future extensibility. Should larger payloads reference an IPFS CID instead of embedding directly?
4. **Forward secrecy** — Is this a v1 requirement or acceptable to defer? A double-ratchet style scheme would be complex but possible.
5. **Group sessions** — Out of scope for v1 but worth designing around. Should the `pid` field support multi-peer session tokens?
6. **Polling vs. webhooks** — AlgoNode and other indexers are exploring WebSocket support. Should BEACON specify a WebSocket mode for lower-latency knock delivery?
7. **ARC number** — When this is ready to formalise, which ARC track is most appropriate?

---

## How to Give Feedback

This is a draft RFC — nothing here is final. Please join the **Discussions** tab to share your thoughts, or open an **Issue** for specific bug reports or spec problems. We are specifically looking for:

- Security review of the encryption scheme
- Thoughts on the protocol address approach
- Use cases this could serve beyond P2P chat
- Anything that seems underspecified or broken

Pull requests to improve the spec are also welcome.

---

## License

This spec is released into the public domain. Implement it however you like.
