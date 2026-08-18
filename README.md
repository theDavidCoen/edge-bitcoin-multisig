# Edge Bitcoin P2WSH multisig

Public reference for Edge developers reviewing **native Bitcoin P2WSH multisig** in Edge React GUI. This work was done by **David Coen** with [Cursor](https://cursor.com) as a prototype/demo (**Edge Grana** APK, `app.edge.grana`) on top of Edge `4.50.0`.

**Share this repo** with Edge staff. Application code lives on the linked GUI branch. The Nostr **coordination protocol** (why encrypted DMs, message types, threat model) is specified separately — see the table.

| Component | Repository | Branch / notes |
|-----------|------------|----------------|
| **React GUI** (create/import/spend UI, BIP-48 keys, Blockbook watch, Nostr client) | [theDavidCoen/edge-react-gui](https://github.com/theDavidCoen/edge-react-gui) | [`multisig`](https://github.com/theDavidCoen/edge-react-gui/tree/multisig) |
| **This document** (Edge implementation) | [theDavidCoen/edge-bitcoin-multisig](https://github.com/theDavidCoen/edge-bitcoin-multisig) | `main` |
| **Coordination protocol** (Nostr layer, not Edge-specific) | [theDavidCoen/nostr-multisig-coordination](https://github.com/theDavidCoen/nostr-multisig-coordination) | `main` |

Upstream base (for merge planning):

- GUI: [EdgeApp/edge-react-gui](https://github.com/EdgeApp/edge-react-gui) @ `4.50.0`

No changes are required in `edge-currency-plugins` for receive/spend of the **shared P2WSH**. The existing Bitcoin bip49 wallet is only a **shell** (UI, seed storage, sync). Shared funds live on a BIP-67 P2WSH script watched via Blockbook from the GUI.

---

## Demo

[~3 min, Android / Edge Grana] Create a 2-of-3 wallet, invite two cosigners by npub, slide to join, and watch statuses update until the shared P2WSH is ready.

**[Watch the full-flow demo](https://github.com/theDavidCoen/edge-bitcoin-multisig/blob/main/demo/full-flow.mp4)** (`demo/full-flow.mp4`)

---

## APK (Edge Grana)

Sideload build, `app.edge.grana`, Edge `4.50.0` + this prototype (18 Aug 2026). ~136 MB; attached as a [GitHub Release](https://github.com/theDavidCoen/edge-bitcoin-multisig/releases) (over GitHub’s 100 MB git-blob limit).

**[Download APK](https://github.com/theDavidCoen/edge-bitcoin-multisig/releases/download/apk-20260818/app-release.apk)** · [Release notes](https://github.com/theDavidCoen/edge-bitcoin-multisig/releases/tag/apk-20260818)

Install over a previous Grana with the same signing key (`adb install -r`). Not on Play Store.

---

## Why this belongs in Edge

1. **From-scratch multisig is an Edge-user product.** New wallets are created by inviting other Edge accounts (Nostr `npub` or NIP-05). That is intentional, not an unfinished xpub form.
2. **Coordination without a coordinator server.** Invites, acceptances, and cosignature requests travel as encrypted Nostr DMs (NIP-17 gift wraps). Relays never see xpubs or PSBTs in plaintext.
3. **On-chain keys stay BIP-48.** The same BIP-39 seed shown as Master Private Key / Raw Keys `bitcoinKey` derives both the bip49 shell and the cosigner account `m/48'/0'/0'/2'`. The descriptor origin is not a cosmetic label — the serialized xpub is depth 4, index `2'`.
4. **Recovery is still open.** BIP-380 descriptors and BIP-129 BSMS can be exported and imported (paste). Watching or restoring an existing script does not require the other cosigners to be Edge users.

---

## 1. Edge-only from-scratch creation (entropy)

Pasting an xpub from another app into **Create** would let that app’s key material into a brand-new script. Some third-party wallets have shipped with **insufficient entropy** (the **Coincard** case). A well-formed xpub from a weak RNG still looks like a valid cosigner. The resulting multisig can be weaker than it appears, including related-key / reduced-search-space failures that are invisible in the UI.

**Policy in this prototype**

| Path | Allowed | Why |
|------|---------|-----|
| Create from scratch | Edge `npub` / NIP-05 only | Every cosigner key is derived inside Edge from a BIP-39 `bitcoinKey` |
| Invite by pasted xpub | **Hidden** (`ENABLE_MULTISIG_XPUB_INVITE = false`) | Avoids importing low-entropy foreign keys at creation time |
| Import descriptor / BSMS | Yes (paste) | Recovery / watch of a script that already exists |
| Export descriptor / BSMS | Yes | Interop with Sparrow, Specter, etc. |

The xpub invite path is **still implemented** in `startMultisigWallet({ mode: 'xpub' })` so it can be re-enabled later (for example behind a developer flag or after an entropy warning). Do not treat the hidden UI as deleted code.

NIP-05 is resolved over HTTPS (`/.well-known/nostr.json`) to an `npub`, then the invite is sent to that `npub`. NIP-05 is not a Bitcoin key and is not written into the on-chain descriptor.

---

## 2. Product model

```
Edge Bitcoin wallet (bip49 shell)
  bitcoinKey  ──►  m/49'/0'/0'     UI / legacy singlesig addresses (not the shared funds)
                └►  m/48'/0'/0'/2'  cosigner xpub in the P2WSH descriptor
```

- **Shell wallet:** `walletType: wallet:bitcoin-bip49`, `keyOptions: { format: 'bip49' }`. Created so Edge has a normal Bitcoin wallet row, seed backup, and password gating.
- **Shared funds:** native segwit P2WSH (`wsh(sortedmulti(m, …))`), BIP-67 pubkey sort, receive at `m/0/i` under each account xpub (descriptor multipath `/<0;1>/*`).
- **Balance / history / receive address** for the shared script come from Blockbook in the GUI (`p2wshWatch.ts`), not from the bip49 engine UTXO set.
- **Nostr** is transport and UX only. Losing the nsec does not lose Bitcoin; losing `bitcoinKey` does.

---

## 3. Architecture overview

```mermaid
flowchart TB
  subgraph GUI["edge-react-gui"]
    Create["CreateWalletMultisigScene<br/>npub / NIP-05"]
    Pending["MultisigPendingScene"]
    Spend["SendScene2 + MultisigSpendPendingScene"]
    Keys["multisigKeys.ts<br/>m/48'/0'/0'/2'"]
    P2WSH["bitcoinP2wsh.ts<br/>descriptor + address"]
    Watch["p2wshWatch.ts"]
    NostrSvc["MultisigNostrService"]
  end

  subgraph Store["account.dataStore edge-multisig"]
    Identity["Nostr nsec / npub"]
    Proposals["proposals + spends"]
    Outbox["pending Nostr outbox"]
  end

  subgraph Chain["Bitcoin"]
    BB["Edge Blockbook"]
    Script["P2WSH bc1q…"]
  end

  subgraph Relays["Nostr relays"]
    GW["kind 1059 gift wraps"]
  end

  Create --> Keys
  Create --> NostrSvc
  NostrSvc --> GW
  NostrSvc --> Proposals
  Keys --> P2WSH
  P2WSH --> Script
  Watch --> BB
  Watch --> Script
  Spend --> P2WSH
  Identity --> NostrSvc
```

---

## 4. GUI map (`src/`)

| Path | Role |
|------|------|
| `actions/MultisigActions.ts` | Create, import, accept/decline, repair BIP-48, spend request / partial / broadcast |
| `actions/MultisigExportActions.tsx` | Password-gated descriptor / BSMS export |
| `components/scenes/CreateWalletMultisigScene.tsx` | m-of-n, npub or NIP-05, confirm screen |
| `components/scenes/CreateWalletImportScene.tsx` | Seed + descriptor/BSMS paste |
| `components/scenes/MultisigPendingScene.tsx` | Join / wait for cosigners |
| `components/scenes/MultisigSpendPendingScene.tsx` | Approve / reject PSBT |
| `components/scenes/NostrAccountScene.tsx` | npub, nsec reveal, NIP-05 display name |
| `components/services/MultisigNostrService.tsx` | Inbox, outbox flush, BIP-48 repair on login |
| `util/multisig/multisigKeys.ts` | Derive BIP-48 from `bitcoinKey` via `deriveChild` (no path-string apostrophe issues on Hermes) |
| `util/multisig/bitcoinP2wsh.ts` | Normalize SLIP-132 keys, BIP-67 script, descriptor build/parse, **refuse to stamp `48'/0'/0'/2'` on a non-BIP-48 xpub** |
| `util/multisig/parseImport.ts` | Descriptor / BSMS parse; seed match prefers real BIP-48 bytes over origin labels |
| `util/multisig/spendPsbt.ts` | Build/sign/finalize PSBT; Blockbook broadcast |
| `util/multisig/p2wshWatch.ts` | Watch receive gap, balance, incoming txs |
| `util/multisig/exportInfo.ts` | Export payload; drop stored descriptors whose origin disagrees with xpub depth/index |
| `util/nostr/*` | Identity, NIP-17, NIP-44, NIP-05 lookup, relay pool, outbox |
| `locales/en_US.ts` | All user-visible copy (English source) |

Create-wallet list: `getCreateWalletList.ts` adds **Bitcoin (Multisig)** with `isMultisig: true`. Mixing Multisig with another asset in the same create flow is rejected (`multisig_cannot_mix_assets`).

---

## 5. Keys (must match the seed UI)

Master Private Key and Raw Keys `bitcoinKey` are the **bip49 shell seed** (12/24 words). That seed **must** also produce the BIP-48 account xpub used on-chain.

| Check | Expected |
|-------|----------|
| BIP-32 depth of local cosigner xpub | `4` |
| Child index | `2'` (`0x80000002`) |
| Path | `m/48'/0'/0'/2'` |
| Same seed, BIP-49 account | `m/49'/0'/0'` — **different** xpub, depth 3 |

`buildMultisigDescriptor` throws if asked to label a non-BIP-48 xpub as `48'/0'/0'/2'`. Import matching uses the derived xpub bytes, not the origin string in the descriptor (old wallets could stamp `48'` on a BIP-49 node).

Fingerprint in `[fp/48'/0'/0'/2']` is the **master fingerprint** when the root is depth 0.

---

## 6. Create / import / spend (Edge UX)

### 6.1 Create

1. Select **Bitcoin (Multisig)** → name → m-of-n.
2. Each other cosigner: paste **npub** or **NIP-05** (`name@domain`, or a bare domain → `_@domain`).
3. **Next** resolves NIP-05 over HTTPS, then loads kind 0 from relays for display names.
4. **Confirm cosigners** shows NIP-05 next to the resolved npub.
5. **Create** makes the bip49 shell, stores a pending proposal, gift-wraps `edge-multisig-invite` to each npub.

The invite JSON (plaintext inside NIP-17) is documented in [nostr-multisig-coordination](https://github.com/theDavidCoen/nostr-multisig-coordination). Edge does **not** put NIP-05 in the wire message — only the resolved npub.

### 6.2 Accept

Invitee sees an in-app notification → pending scene → slide to join. Edge derives their BIP-48 xpub from **their** shell seed and publishes `edge-multisig-accept`. When every slot has an xpub, the initiator publishes `edge-multisig-complete` and all clients derive the same P2WSH.

### 6.3 Import

Paste BIP-380 `wsh(sortedmulti(…))` or wallet BSMS 1.0 plus the local seed. Signer-only BSMS (`00` + one xpub) is rejected. The seed must match one cosigner (BIP-48 preferred, legacy BIP-49 allowed for old scripts).

### 6.4 Spend

Send from the shell wallet row is intercepted: GUI builds a P2WSH PSBT, partial-signs locally, gift-wraps `edge-multisig-spend-request`. Other devices sign (`-partial`) or reject. When `m` signatures exist, the last required signer finalizes and broadcasts via Blockbook.

Legacy BIP-49 cosigner keys (pre-BIP-48 wallets) still sign through `legacyBip49` in `spendPsbt.ts`.

---

## 7. Swap support status

This prototype includes the **multisig swap UX and spend-flow wiring**, but
local testing currently has a hard limitation:

- The Grana / local dev build in this repo does **not** have access to the
  production exchange-provider secrets.
- In Edge, those secrets are **not versioned in git** on `master`; production
  pulls them in from an external secret-files source during build time.
- Without that production `env.json` content, Bitcoin swap providers either do
  not initialize fully or cannot be validated locally, so BTC swaps return
  **“No enabled exchanges support BTC to ETH”** even for a normal singlesig
  Bitcoin wallet.

This means the local error is **not evidence that the multisig PSBT / Nostr
flow is wrong**. It only means this prototype cannot fully exercise the
provider side of swap quote creation from a developer build that lacks the
production exchange setup.

During QA, a few app-side compatibility shims were tested and then removed
again. Those experiments should **not** be treated as the intended final
integration path. The real unresolved dependency is still access to the
production exchange-provider setup.

### What developers should do when wiring production swap testing

Once Edge developers test with the real production exchange configuration,
multisig swaps should be implemented with this contract:

1. **Treat the multisig wallet like Bitcoin for provider compatibility.**
   Quote / approval should be requested as if the asset were a normal Bitcoin
   wallet so centralized swap providers can price BTC pairs normally.
2. **Use a P2WSH address from the multisig wallet anywhere the provider expects
   a wallet address.**
   That includes the address surfaced to the provider as the wallet’s
   destination / refund / return address.
3. **Keep the actual send path multisig-native.**
   After the provider returns a deposit address, Edge must still build a P2WSH
   PSBT, send cosignature requests over encrypted Nostr, and only broadcast
   once the multisig threshold is met.
4. **Preserve the 5-minute expiry for swap PSBTs.**
   If cosigners do not complete the request in time, the swap request should be
   cancelled and forgotten.

In short: **providers should see “plain Bitcoin”, but the wallet address they
interact with must be a P2WSH address belonging to the multisig wallet, and
Edge must keep the actual spend flow on the PSBT/Nostr multisig path.**

---

## 8. Buy and Sell notes

These flows are **not** fully implemented in this prototype, but they are the
next logical product/design questions once swap-provider access is available.

### 8.1 Buy

`Buy` should be the simpler case:

- The provider should request a normal receive address from the selected wallet.
- For a complete multisig wallet, Edge should surface a **P2WSH receive
  address** from the multisig wallet, not a bip49 shell address.
- The provider can then send purchased BTC directly to that P2WSH address.
- No cosigner interaction is required, because this is an **incoming** flow.

So the mental model for developers is:

> `Buy` for multisig should behave like ordinary Bitcoin receive, except the
> receive address exposed to the provider must come from the shared P2WSH
> wallet.

### 8.2 Sell

`Sell` needs product and technical review.

Today many sell flows assume a normal single-signer “slide to confirm” send.
For multisig, developers should instead evaluate whether that final confirmation
must become the same signing model already used for multisig send:

1. Build the spend as a **PSBT** against the multisig P2WSH UTXOs.
2. Send the spend request to cosigners over **encrypted Nostr**.
3. Collect signatures until the threshold is met.
4. Let the last required signer finalize and broadcast.

Open questions for Edge developers:

- Can the provider flow tolerate replacing **slide to confirm** with
  **slide to sign**?
- At what point in the sell UX should the provider order be considered locked
  if broadcast still depends on cosigner participation?
- Should sell requests use the same **5-minute maximum window** already used
  for multisig swap PSBTs, after which the request is cancelled and forgotten?
- If the provider has stricter timing guarantees, does sell need a distinct
  expiry model from swap/send?

Recommended starting point:

> Treat `Sell` as a multisig spend first, and only then map the provider UX on
> top of that constraint. If the provider requires an unconditional immediate
> broadcast after one local gesture, then the standard singlesig sell UX is not
> directly compatible with a threshold wallet.

---

## 9. Persistence

Store id: `edge-multisig` on `account.dataStore`.

| Key | Contents |
|-----|----------|
| `identity` | nsec hex, npub, optional display name / NIP-05 |
| `proposals` | Create/join state, cosigner xpubs, descriptor, P2WSH address |
| `spends` | In-flight PSBTs and signer status |
| `onchain-{walletId}` | Compact on-chain snapshot for watch |
| `pendingNostrOutbox` | Gift wraps to retry if a relay publish fails |

The Nostr secret is **not** derived from `bitcoinKey`. It is created with `schnorr.utils.randomSecretKey()` on first **multisig/Nostr use** (`ensureNostrIdentity` from Create Multisig or Settings → Nostr Account), not on every login. Users can replace it from Settings → Nostr Account by importing an `nsec`.

---

## 10. Relays and Blockbook

Default Nostr relays (`util/nostr/relayPool.ts`):

- `wss://relay.damus.io`
- `wss://nos.lol`
- `wss://relay.primal.net`

Inbox is kind **1059** gift wraps tagged `p` = this device’s pubkey. Profiles are kind **0** via a one-shot `queryRelays` so they do not disturb the inbox subscription.

`MultisigNostrService` must **not** auto-create an identity or open these relays on login for accounts that have never used multisig. That raced wallet sync with ~9 WebSockets (inbox pool + kind-0 queries) and made the whole app feel stuck. Subscribe only once an identity already exists; create keys lazily in Create Multisig / Nostr settings.

P2WSH watch uses the same Edge Blockbook bases as Bitcoin (`getBlockbookBases`).

---

## 11. Tests

```bash
cd edge-react-gui
npm test -- --testPathPatterns='multisig|nip05|nostrMultisig' --watchAll=false
```

Coverage includes BIP-48 depth/index vs origin labels, descriptor checksums, import matching, NIP-05 well-known lookup, NIP-17 wrap/unwrap, and PSBT 2-of-3.

---

## 12. Grana release APK (local QA)

```bash
export ANDROID_HOME=/home/david/.android-sdk
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export GRADLE_USER_HOME=/home/david/.gradle
cd edge-react-gui/android
./gradlew assembleRelease -PreactNativeArchitectures=arm64-v8a
# APK: app/build/outputs/apk/release/app-release.apk
# applicationId: app.edge.grana
```

Not a production Edge Play Store build.

---

## 13. Performance — progress and remaining work

The join/complete protocol works, and some of the worst early latency issues
have already been improved. The remaining items below are the ones that still
need product-level attention.

### 13.1 Improved in this prototype

1. **Invite card after login** is in better shape than the earliest versions.
   The app no longer auto-creates a Nostr identity or opens the relay set on
   login for accounts that have never used multisig. Cached joinable invites
   can also surface before full relay repair/cleanup finishes.

2. **Wallet list vs P2WSH balance** is improved. The shell bip49 balance is no
   longer the only balance users see for a complete multisig wallet. The GUI
   now surfaces the shared P2WSH balance in the wallet list, balance card, and
   swap entry points so the shell wallet is less likely to be mistaken for the
   real funds.

3. **Relay startup behavior** is improved. Subscribe/poll behavior is more
   conservative, the first inbox poll uses a short timeout, and retry timing is
   more aggressive than the original “wait for a full reconnect cycle” model.

### 13.2 Still open

1. **Loading after slide to join.** `acceptMultisigInvite` still does too much
   on the slider critical path: orphan cleanup, BIP-49 wallet create/resolve,
   BIP-48 key derivation, persist, Nostr accept/complete publish, notification
   bookkeeping, and orphan replay. The slider should still get lighter so the
   UI can leave the pending scene in ~1–2 seconds and finish non-essential work
   in the background.

2. **Cosigner status after join.** Local state is better than before, but the
   remote-peer round trip is still not ideal. Pending proposals still rely on a
   periodic inbox poll, so accept/complete propagation can remain slower than a
   truly responsive product experience.

### 13.3 Do not regress

Do **not** “fix” latency by restarting the Nostr subscribe effect, stopping
the pool, or clearing the orphan inbox on unrelated wallet-load updates —
that reintroduces the long invite-card delay.

Do **not** auto-create a Nostr identity or open Damus / nos.lol / Primal
sockets on login for accounts that have never used multisig.

---

## 14. Suggested merge checklist for Edge

- [ ] Security review: Nostr nsec in `dataStore`, NIP-17 to public relays, Blockbook for P2WSH
- [ ] Confirm BIP-48 derivation from `bitcoinKey` (depth 4 / index `2'`) on device, not only unit tests
- [ ] Product sign-off: Edge-only from-scratch invites (Coincard / entropy rationale)
- [ ] Decide whether xpub-create returns behind a warning, never, or export-only
- [ ] Import/export interop: Sparrow / Specter descriptor + BSMS
- [ ] Copy review for pending invite, spend request, and receive (P2WSH ≠ bip49 address)
- [ ] Hermes: BIP-48 uses `deriveChild`, not a path string containing `'`
- [ ] Performance: invite card ~2s after login; slide-to-join not blocked on Nostr/wallet work
- [ ] Performance: no Nostr sockets / identity on login for accounts that never used multisig

---

## 15. Contact

**David Coen** · prototype for Edge review (built with Cursor).

On-chain scripts follow BIP-48, BIP-67, BIP-174/370 (PSBT), BIP-380, BIP-129. Coordination messages are specified in [nostr-multisig-coordination](https://github.com/theDavidCoen/nostr-multisig-coordination).
