# BEACON Protocol
### Blockchain-Encrypted Algorand Connection Over Note-field

**Status:** RFC — Request for Comments  
**Version:** 0.2  
**Author:** [@loafpickleWW](https://github.com/loafpickleWW)  
**Discussion:** Join the Discussions tab or open an issue on this repo

---

## Summary

BEACON is a serverless, privacy-preserving signaling protocol that uses the Algorand note field to bootstrap peer-to-peer WebRTC connections between wallets — with no brokers, no relays, no third-party infrastructure, and no on-chain relationship metadata between communicating parties.

Any Algorand address is natively a BEACON recipient. No key registration, no separate identity infrastructure, no NFD required. The recipient's wallet keypair is the only thing that can decrypt a message addressed to them — enforced by mathematics, not policy.

---

## Motivation

Establishing a direct encrypted connection between two parties today requires a signaling server — a broker that helps both sides exchange WebRTC session descriptions before the peer-to-peer channel opens. This is a meaningful point of centralisation: the broker knows who is connecting to whom, when, and from where. 

Furthermore, this reliance creates 3rd-party vendor lock-in. Traditional infrastructure requires someone to pay for hosting, maintain servers, and ensure uptime. If the company hosting the signaling server shuts down or changes their terms, your application breaks.

Algorand's note field is an underutilised primitive. At 1KB, it is large enough to carry an encrypted connection offer. At 0.001 ALGO per transaction, it is cheap enough to use as ephemeral signaling infrastructure. By replacing proprietary servers with a public, decentralized blockchain, we can build open protocols that never go down, cannot be censored, and belong to no single vendor.

BEACON replaces the signaling broker entirely.

---

## Core Design

### The Protocol Address

BEACON uses an open, shared protocol address as a public noticeboard. The protocol is completely open, so anyone can configure their client to monitor any address they choose. All BEACON traffic for a shared instance — offers, answers, pings, rejections — is sent to this address as 0 ALGO transactions with encrypted note fields.

**How Crypto Solves Vendor Lock-in:** 
By using a shared protocol address on a public ledger, BEACON removes the need for hosted backends. The blockchain acts as an immutable, globally available message bus. Developers don't have to worry about API rate limits from private companies, recurring server costs, or infrastructure deprecation. The network provides the service, paid for per-use (fractions of a cent) directly by the users, ensuring the signaling layer will outlive any individual application or company.

Every BEACON client polls this address for new transactions. Decryption is the inbox filter: if a note decrypts successfully with your private key, it is addressed to you. If it does not, it is silently ignored.

**Privacy properties of this design:**

- The sender's address is visible on-chain. The recipient's address never appears.
- Note content is encrypted — an observer learns nothing about intent, recipient, or outcome.
- All clients poll the same address, so polling behaviour reveals nothing about who you are expecting to hear from.
- There is no on-chain link between any two communicating wallets.
- The WebRTC session token is never present in the note — it is derived from the shared secret out of band.

### Encryption Model

Algorand accounts are ed25519 keypairs. The public key is mathematically embedded in every Algorand address. BEACON converts this ed25519 public key to a curve25519 key using the well-defined Bernstein conversion, enabling NaCl box encryption directly to any Algorand address with no prior key exchange or registration step.

```
algoAddress
  → decode base32 → extract 32-byte ed25519 public key
  → convert to curve25519 (Bernstein conversion)
  → NaCl box encrypt
```

This means any Algorand address is automatically a valid BEACON recipient. The sender needs nothing beyond the recipient's address to encrypt a message that only the recipient's wallet can decrypt.

### Key Derivation

```javascript
import { convertPublicKey } from "ed2curve";

function algoAddressToCurve25519(address) {
  const decoded = algosdk.decodeAddress(address);
  return convertPublicKey(decoded.publicKey);
}
```

The `ed2curve` library implements the Bernstein conversion. `tweetnacl` handles the NaCl box encryption. Both are small, audited, and widely deployed.

### Authentication via Simulate

Rather than requiring a separate wallet prompt for every BEACON poll, recipient authentication uses Algorand's simulate endpoint — a dry-run transaction evaluation that requires a valid signature but never broadcasts to the network and costs nothing.

```
1. Client constructs a zero-cost transaction with a random nonce in the note field
2. Submits to /v2/transactions/simulate — no broadcast, no fee, no user prompt
3. Simulate confirms the signature is valid for the claimed address
4. Client proceeds to attempt decryption
```

This proves the client controls the private key for a given address passively, without requiring active wallet approval on every poll cycle. It is the same proof-of-ownership mechanism used in BEACON's existing handshake flow, extended to work silently in the background.

### Session Token Derivation

The WebRTC peer ID never appears in the note payload. Instead, both sides independently derive a one-time session token from the shared secret established during the NaCl key exchange:

```javascript
const sharedSecret = nacl.box.before(senderPubkey, recipientSecretKey);
const sessionToken = blake2bHex(
  concat(sharedSecret, toBytes(offer.ts))
).slice(0, 32);
```

Both sides compute the same token without communicating it. Alice registers as a WebRTC peer with `id = sessionToken`. Bob derives the same token and connects to it. The peer ID never touches the chain and never appears in any log in a form linkable to either wallet.

The exact `ts` value from the offer note is used in derivation — not a rounded timestamp — so both sides converge on exactly the same token without ambiguity.

---

## Note Structure

All notes are prefixed with `BEACON/1:` for indexer filtering, followed by a base64-encoded NaCl box:

```
BEACON/1:<base64(nacl.box(JSON, nonce, recipientCurve25519Key, ephemeralSecretKey))>
```

The decrypted JSON payload:

```json
{
  "proto": "BEACON/1",
  "type": "offer",
  "pk": "<sender ephemeral curve25519 pubkey, base64>",
  "ts": 1718000000,
  "exp": 1718003600
}
```

| Field | Description |
|---|---|
| `proto` | Protocol version string |
| `type` | Message type (see below) |
| `pk` | Sender's ephemeral curve25519 pubkey — recipient uses this to derive the shared secret and compute the session token |
| `ts` | Unix timestamp of issue — used in session token derivation |
| `exp` | Unix timestamp of expiry — clients must ignore stale messages |

Note the absence of a `pid` field. The session token is derived, never transmitted.

### Message Types

| Type | Direction | Description |
|---|---|---|
| `offer` | A → noticeboard | Initiator announces session availability |
| `answer` | B → noticeboard | Responder confirms readiness |
| `reject` | B → noticeboard | Explicit decline, encrypted so only initiator sees it |
| `ping` | Any → noticeboard | Presence beacon, no session implied |
| `revoke` | A → noticeboard | Cancel a pending offer before expiry |

---

## Full Session Flow

```
Alice                    Algorand (BEACON Address)              Bob
  │                                │                              │
  │  1. Derive curve25519 pubkey   │                              │
  │     from Bob's algo address    │                              │
  │                                │                              │
  │  2. Generate ephemeral keypair │                              │
  │     Encrypt offer to Bob's     │                              │
  │     curve25519 key             │                              │
  │     Note contains only:        │                              │
  │     ephemeral pubkey + ts/exp  │                              │
  │                                │                              │
  │──── 0 ALGO + BEACON note ─────▶│                              │
  │                                │                              │
  │                                │◀──── polling every N secs ───│
  │                                │      simulate auth           │
  │                                │      try decrypt all notes   │
  │                                │                              │
  │                                │    Bob's key decrypts ✓      │
  │                                │    derive sharedSecret from  │
  │                                │    ephemeral pubkey          │
  │                                │    derive sessionToken       │
  │                                │    from sharedSecret + ts    │
  │                                │                              │
  │                                │    Bob encrypts answer       │
  │                                │    to Alice's curve25519 key │
  │                                │    (derived from her address)│
  │                                │                              │
  │◀──── polling every N secs ─────│◀─── 0 ALGO + BEACON note ───│
  │  Alice's key decrypts ✓        │                              │
  │  derive same sessionToken      │                              │
  │  from shared secret + ts       │                              │
  │                                │                              │
  │  Alice registers WebRTC        │    Bob connects to           │
  │  peer as sessionToken          │    sessionToken peer ID      │
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

`lastSeenRound` is persisted in localStorage and updated after each successful poll. Clients never reprocess old traffic. The fixed address and shared note prefix makes this query highly cache-friendly at the indexer level — every BEACON client in existence makes an identical request.

**Suggested polling intervals:**

| Client state | Interval |
|---|---|
| Active / listening | 5 seconds |
| Idle / background | 30 seconds |
| Offer pending | 5 seconds until answer or expiry |

---

## Security Considerations

**Ed25519 to curve25519 key conversion**
Reusing a signing keypair for encryption is a practice that warrants explicit review. The Bernstein conversion is mathematically sound and used in production protocols. However, if the curve25519 private key were derivable from a compromised signing key, past encrypted messages could be exposed. Implementations should treat the converted key as equally sensitive to the signing key. Community review of this specific aspect is particularly welcomed.

**Replay attacks**
The `exp` field provides expiry. Clients must reject any message where `exp` has passed or where `ts` is more than 60 seconds in the future (clock skew tolerance).

**Spam**
The 0.001 ALGO minimum transaction fee provides natural rate limiting. Client-side filtering by drop rate per sender address per round is recommended for resilience.

**Sender anonymity**
The sender's Algorand address is visible on-chain. For stronger sender anonymity, clients may use a throwaway session wallet funded with a small amount of ALGO. The protocol does not mandate this but implementations should make it easy.

**Forward secrecy**
Per-session ephemeral keypairs provide partial forward secrecy at the signaling layer — compromise of a long-term key does not expose past session tokens. Full forward secrecy at the communication layer depends on the WebRTC implementation. This is a known limitation of v1.

**Rekeyed accounts**
Algorand supports rekeying an account to a different spending key. In a rekeyed account the address-derived ed25519 key and the actual signing key diverge. BEACON encryption targets the address-derived key — a rekeyed account holder can still decrypt using the original private key even if they sign transactions with a different key. Implementations should document this clearly.

**Protocol address**
The protocol address is unowned and unspendable. No party can censor the noticeboard or selectively block traffic.

---

## Open Questions for Discussion

These are the areas where community input would be most valuable:

1. **Protocol address selection** — Should this be a known burn address, a vanity address, or a multisig with no valid signers? What are the tradeoffs?

2. **Ed25519 → curve25519 conversion** — Is the community comfortable with this approach? Are there Algorand-specific concerns beyond the general cryptographic arguments?

3. **Simulate as passive auth** — Are there edge cases where simulate-based authentication could be spoofed or bypassed? Particularly interested in rekeyed accounts and multisig wallets.

4. **Session token derivation** — Is BLAKE2b the right choice here? Should the derivation include additional entropy beyond shared secret and timestamp?

5. **Note size constraints** — 1KB is sufficient for v1 but limits future extensibility. Should larger payloads reference an IPFS CID instead of embedding directly?

6. **Forward secrecy** — Is this a v1 requirement or acceptable to defer?

7. **Multisig accounts** — How should BEACON handle multisig addresses where no single party holds the full private key?

8. **ARC track** — When ready to formalise, which ARC track is most appropriate?

---

## Dependencies

| Library | Purpose |
|---|---|
| `algosdk` | Address decoding, transaction construction, simulate |
| `tweetnacl` | NaCl box encryption and decryption |
| `ed2curve` | Ed25519 to curve25519 key conversion |
| `@noble/hashes` or equivalent | BLAKE2b session token derivation |

All are small, audited, and have no server-side requirements.

---

## Reference Implementation

BEACON/1 is currently implemented in [Wen Tools P2P Chat](https://wentools.xyz) — a serverless, wallet-authenticated encrypted chat and file transfer tool built on Algorand. The implementation uses `@txnlab/use-wallet-react` for wallet connection, `tweetnacl` for encryption, and the AlgoNode public indexer for polling (no API key required).

---

## How to Give Feedback

This is a draft RFC — nothing here is final. Open an issue on this repo to discuss any aspect of the protocol. Specifically looking for:

- Security review of the ed25519 → curve25519 conversion in this context
- Review of the simulate-based authentication approach
- Thoughts on the session token derivation scheme
- Any Algorand-specific edge cases (rekeyed accounts, multisig, etc.)
- Use cases beyond P2P chat this design could serve

Pull requests to improve the spec are welcome.

---

## Changelog

| Version | Changes |
|---|---|
| 0.2 | Replaced curve25519 + NFD key registry with native ed25519→curve25519 address derivation. Added simulate-based passive authentication. Removed `pid` from note payload in favour of derived session tokens. Clarified rekeyed account behaviour. |
| 0.1 | Initial draft |

---

## License

This spec is released into the public domain. Implement it however you like.
