# AZNS — ENS sunrise (priority `.tru` for `.eth` owners) (design spec)

> Status: **proposal / not implemented.** Pre-audit, pre-mainnet. Adds net-new
> cross-chain + signature-verification surface to an unaudited contract; treat
> as a design to slot in deliberately, not a rush job. Expands audit scope.

## Goal

Give existing **ENS name owners a head-start** on the matching `.tru` name. For a
bounded **sunrise window** (e.g. 3 weeks from launch), the owner of `vitalik.eth`
is the only party who may register `vitalik.tru`. When the window closes,
registration reverts to today's fully permissionless behaviour — anyone may
claim any still-unregistered name.

This is the standard TLD-launch **sunrise period** (what ICANN runs off the
Trademark Clearinghouse), applied to a privacy L2 name service.

## Where the hard part is (not the timer)

The window itself is trivial — AZNS already has all the lifecycle machinery
(`register` at [main.nr:212](../azns/src/main.nr), `_finalize`'s availability
check at [main.nr:291](../azns/src/main.nr), `self.context.timestamp()`, a
90-day grace concept). Gating registration on a clock is a few asserts.

The entire difficulty is **proving ENS ownership to AZNS**: ENS ownership is
Ethereum-L1 state, AZNS is a private Aztec contract that cannot read L1. Every
real decision in this doc is about *how much trust* that proof carries. So this
spec is 10% window mechanics, 90% the proof.

---

## Lifecycle: the sunrise window

One immutable timestamp, set at deploy, splits registration into two paths:

```
                       sunrise_deadline
   deploy ────────────────────┼──────────────────────────►  time
        │   SUNRISE WINDOW     │      GENERAL AVAILABILITY
        │                      │
 register()              ──►  reverts             ──►  permissionless (today's
   (permissionless)            "sunrise: ENS            behaviour, unchanged)
                               owners only"
 register_with_ens_proof() ──► allowed, needs    ──►  still allowed but pointless
                               a valid ENS proof       (GA path is open anyway)
```

- **During the window:** the plain `register` is disabled; the only way in is
  `register_with_ens_proof`, which requires a proof that the caller owns the
  matching `.eth` name.
- **After the window:** `register` is open exactly as it is today. "Grace
  elapses → anyone can register" falls out for free — no new state.

Everything else in AZNS (pricing, modes, lease/grace, epochs) is untouched. A
sunrise registration still pays the normal per-mode fee and still flows through
`_finalize`, so it's an ordinary lease from that moment on.

---

## Two designs for the ENS-ownership proof

Both keep AZNS's privacy intact: verification happens **inside the private
`register` proof**, so the only thing that becomes public is the usual
`name_hash → owner` write in `_finalize` ([main.nr:306](../azns/src/main.nr)) —
identical to a normal registration. The L1↔Aztec linkage never hits the chain.

### Option A — Trusted attester voucher (MVP)

An off-chain **attester** service reads ENS on L1 and issues signed vouchers.

```mermaid
sequenceDiagram
    participant U as Claimant (owns vitalik.eth)
    participant W as MetaMask (L1 key)
    participant A as Attester service
    participant C as AZNS (Aztec, private)
    U->>A: request voucher for "vitalik", aztec_owner=0xAZ...
    A->>U: challenge nonce
    U->>W: personal_sign(challenge ‖ aztec_owner)
    W->>A: L1 signature
    A->>A: ecrecover == ENS owner(namehash("vitalik.eth"))?
    A->>U: voucher = Schnorr_sig_attester(name_hash, aztec_owner, deadline)
    U->>C: register_with_ens_proof(..., voucher)
    C->>C: verify attester Schnorr sig; charge fee; _finalize
```

**Voucher (what the attester signs):**

```
msg   = poseidon2([DOMAIN_SUNRISE_V1, name_hash, aztec_owner, voucher_deadline])
sig   = Schnorr_sign(attester_sk, msg)          // Grumpkin/poseidon — native to Aztec
```

- `name_hash` is the AZNS name hash `poseidon2([label, label_len])` — the
  attester normalizes the ENS label and computes it exactly as
  [main.nr:250](../azns/src/main.nr) does, so it binds to the very name being
  registered.
- `aztec_owner` binds the voucher to one Aztec account (must equal
  `self.msg_sender()`), so a leaked voucher can't be redeemed by someone else.
- `voucher_deadline` bounds replay to a short redemption window.

**Circuit cost:** one **Schnorr-over-Grumpkin** verification against a
poseidon digest — *native and cheap* on Aztec. The expensive part (ecrecover of
an Ethereum `personal_sign`) happens **off-chain in the attester**, so
`register_with_ens_proof` is barely more expensive than `register`.

**Trust:** the attester is trusted to (a) reflect ENS honestly and (b) bind the
right Aztec owner. Mitigations: run it as **k-of-n multisig** (circuit verifies k
distinct Schnorr sigs); publish the ENS snapshot + rules so anyone can audit
issued vouchers; keep the window short. Because the attester key only matters
*during* the window, storing it immutably is acceptable.

**Ship this for the actual launch.** A one-time, time-boxed sunrise is exactly
where a trusted attester is reasonable, and it costs almost nothing on-chain.

### Option B — Merkle snapshot + in-circuit ECDSA (trust-minimized)

Remove the attester from the claim path; trust only a one-time, reproducible
snapshot of public ENS data.

**Snapshot (built once, off-chain, at a public cutoff block):**

```
for each eligible .eth name:
    label, label_len = normalize(ens_name)         // ENSIP-15 → AZNS 3..31 ASCII
    name_hash        = poseidon2([label, label_len])
    l1_owner         = ENS owner at cutoff block    // see "ENS owner" below
    leaf             = poseidon2([name_hash, l1_owner_as_field])
root = merkle_root(sorted leaves)                   // published immutably on AZNS
```

Keying leaves by the **AZNS `name_hash` (poseidon)**, not the ENS keccak
namehash, is the important trick: it lets the circuit check membership of the
*same* hash it's registering with **no keccak in-circuit** for the name binding.

**Claim — the registrant supplies as private witnesses:**

1. a Merkle proof that `poseidon2([name_hash, l1_owner])` ∈ `root`;
2. an **ECDSA-secp256k1** signature by `l1_owner` over a message committing to
   `aztec_owner` (proves control of the L1 key);
3. (already in `register`) `name_hash == poseidon2([label, label_len])`.

**Circuit cost — honest:** Merkle membership (poseidon, cheap) **+ keccak256 of
the signed message** (Ethereum signs keccak digests, so this is unavoidable with
a normal wallet; ~expensive) **+ `ecdsa_secp256k1::verify_signature`
(expensive)**. Real proving cost, but well-trodden — Aztec's own ECDSA account
contracts do secp256k1-in-circuit.

**Trust:** only the snapshot root, which is deterministically rebuildable from
public L1 state at the cutoff block. No live attester.

### Recommendation

**Launch on A, migrate to B only if you want a permanently trustless sunrise.**
A is days of work and near-zero on-chain cost; B is real circuit work
(keccak+ecdsa) for a property (no trusted attester) that a one-time launch event
doesn't strictly need. The two share the *same* `register_with_ens_proof`
entrypoint shape — the proof payload differs, so B is a drop-in later.

---

## Key design decisions

**Global vs. per-label sunrise.** Cleanest MVP: during the window **every**
registration needs an ENS proof (the whole namespace is reserved for `.eth`
owners), then GA opens. Honoring the literal ask — only *matched* labels locked,
unmatched labels open *immediately* — requires proving a label is **not** in the
ENS set (a non-membership proof). That's feasible (Aztec's nullifier tree is an
indexed Merkle tree built for exactly this) but it's extra circuit work and a
sorted snapshot. **Start global.**

**Fee policy.** ENS owners pay the **normal** per-mode fee (the feature is
*priority*, not *free*). Optional knobs: a sunrise discount, or a longer minimum
lease. Default: no change to pricing.

**Who is the "ENS owner".** Ambiguous on L1: registrant vs. controller vs.
NameWrapper (ERC-1155) owner vs. resolved address. Pick one crisp rule —
recommended: **the NameWrapper/registrar owner of an unexpired 2LD at the cutoff
block.** Explicitly exclude subdomains (`x.vitalik.eth`) and expired/in-grace
`.eth` names.

**Normalization / eligibility.** ENS allows unicode/emoji/1-char/digit names
(ENSIP-15); AZNS labels are **3–31 ASCII**, poseidon-hashed. Define an
eligibility filter: an `.eth` name is sunrise-eligible **iff** it normalizes to a
valid AZNS label. Everything else simply has no `.tru` equivalent and is
unaffected.

**Snapshot cutoff.** Any snapshot has a timestamp; a name transferred after the
cutoff maps to the stale owner. Publish the cutoff block prominently.

---

## Concrete contract changes (Noir)

Grounded in the existing dialect (`#[external]` / `#[initializer]` /
`#[only_self]` / `self.storage` / `enqueue_self` / `poseidon2_hash`).

**Storage additions:**

```rust
// Set once at deploy. During [deploy, sunrise_deadline) plain register() is
// closed and only register_with_ens_proof() admits new names.
sunrise_deadline: PublicImmutable<u64, Context>,
// Option A: attester public key(s). Option B: snapshot merkle root.
attester_pk_x: PublicImmutable<Field, Context>,
attester_pk_y: PublicImmutable<Field, Context>,
// ens_snapshot_root: PublicImmutable<Field, Context>,   // Option B instead
```

**Constructor** gains `sunrise_deadline` + the attester key (or root). A
`sunrise_deadline` of `0` disables sunrise entirely (deploy straight to GA).

**`register` gains one guard** ([main.nr:212](../azns/src/main.nr)):

```rust
// During the sunrise window the permissionless path is closed.
assert(
    self.context.timestamp() >= self.storage.sunrise_deadline.read(),
    "sunrise: use register_with_ens_proof",
);
```

**New entrypoint** — identical charge + label-backup + `_finalize` flow as
`register`, with the proof check bolted on:

```rust
#[external("private")]
fn register_with_ens_proof(
    name_hash: Field, label: Field, label_len: u32,
    owner: AztecAddress, duration_years: u64, mode: u8, fee_nonce: Field,
    // ---- Option A voucher ----
    voucher_deadline: u64,
    sig_s: Field, sig_e: Field,          // attester Schnorr signature
    // ---- Option B would replace the two lines above with:
    // l1_owner: Field, merkle_path: [Field; DEPTH], leaf_index: Field,
    // ecdsa_sig: [u8; 64], ecdsa_pk_x: [u8; 32], ecdsa_pk_y: [u8; 32],
) {
    assert(owner.eq(self.msg_sender()), "owner must be the caller");
    // Same length / duration / mode / name_hash-binding asserts as register().
    // ... (copy main.nr:238-252) ...

    // Only meaningful inside the window (harmless after, GA is open anyway).
    assert(self.context.timestamp() < voucher_deadline, "voucher expired");

    // ---- Option A: verify the attester voucher ----
    let msg = poseidon2_hash([DOMAIN_SUNRISE_V1, name_hash, owner.to_field(), voucher_deadline as Field]);
    let ok = schnorr::verify_signature(
        self.storage.attester_pk_x.read(), self.storage.attester_pk_y.read(),
        [sig_s, sig_e], msg,
    );
    assert(ok, "invalid ENS sunrise voucher");

    // Charge + label backup + finalize — unchanged from register().
    let amount = cents_to_tokens(price_for_mode(mode) * duration_years, self.storage.unit_per_cent.read());
    Token::at(self.storage.payment_token.read())
        .transfer_in_private(owner, self.storage.treasury.read(), amount, fee_nonce)
        .call(self.context);
    self.storage.labels.at(owner).insert(LabelNote::new(label)).deliver(MessageDelivery::onchain_constrained());
    self.enqueue_self._finalize(name_hash, owner, duration_years * YEAR_SECS, mode);
}
```

**`_finalize` is unchanged** — its availability check
([main.nr:298](../azns/src/main.nr)) already prevents a second claim of a name
one ENS owner took, and one snapshot has one owner per name, so no extra
nullifier is needed for uniqueness.

---

## The attester service (Option A)

A small stateless signer, ideally an **AWS Nitro / TEE or an n-of-m multisig**.

1. **Ownership challenge.** Claimant requests a voucher for `label` + their
   `aztec_owner`. Service returns a random challenge. Claimant returns
   `personal_sign(DOMAIN ‖ challenge ‖ aztec_owner)` from their L1 wallet.
2. **Verify.** `ecrecover` the L1 signer; check it equals the chosen "ENS owner"
   (NameWrapper/registrar owner) of `namehash(label + ".eth")` at/after the
   cutoff, and that the name is unexpired and sunrise-eligible (normalizes to a
   valid AZNS label).
3. **Issue.** Normalize → `name_hash`, sign
   `poseidon2([DOMAIN_SUNRISE_V1, name_hash, aztec_owner, deadline])` with the
   attester Schnorr key. Short deadline (e.g. 1h).
4. **Rate-limit + log.** One outstanding voucher per (name, owner); append-only
   log of issued vouchers for public audit against the snapshot.

**Key management / kill switch.** Multisig k-of-n so a single key leak can't mint
vouchers. The window is short, so a compromised key has bounded blast radius; if
needed, the whole feature self-disables at `sunrise_deadline` with no action.

---

## Privacy analysis

- **No new public linkage.** Proof verification is inside the private `register`
  proof. On-chain, a sunrise claim looks like any registration:
  `name_hash → owner` in `_finalize`, plus the usual private fee note + encrypted
  label backup. The fact that `owner` also controls some `.eth` name is **not**
  revealed.
- **Option A:** the L1↔Aztec link is seen only by the attester (off-chain).
- **Option B:** the L1 owner address + Merkle path are circuit **witnesses**,
  never published.
- **Residual:** `.tru` names are public strings once registered (as today), and
  an observer who *independently* knows `vitalik.eth ↔ 0xAZ` can infer the link.
  Sunrise doesn't worsen this; it's inherent to owning a public name.

---

## Security considerations

| Risk | Mitigation |
|---|---|
| Attester compromise (A) mints fake vouchers | k-of-n multisig; short window; public issuance log vs. snapshot; feature auto-expires |
| Voucher / signature replay | Bind `aztec_owner` + `voucher_deadline` into the signed message; `_finalize` single-claim check |
| Snapshot staleness / post-cutoff transfer | Publish cutoff block; accept as the standard sunrise trade-off; B can re-snapshot |
| Two claimants for one name | One ENS owner per name in the snapshot; availability check blocks the second |
| Name-definition confusion (wrapped vs. registrant) | Pin one crisp rule in the attester + snapshot builder; document it |
| `personal_sign` phishing to mint a voucher for attacker's aztec addr | Message must commit to the claimant's own `aztec_owner`; show it in the signing UI |
| Sunrise never opening (bad deadline) | `sunrise_deadline` is immutable and in the past-bounded future; verify at deploy; `0` = no sunrise |

---

## Edge cases

- **ENS name expires mid-window** → still claimable if it was valid at the
  cutoff (snapshot-based) — decide and document whether that's desired.
- **Owner sold the `.eth` after cutoff** → old owner can still claim `.tru`
  (snapshot trade-off).
- **Same label, different TLD intent** (`vitalik.eth` vs. someone who wants
  `vitalik.tru` unrelated) → the ENS owner wins during sunrise; the other waits
  for GA. This is the whole point.
- **Non-eligible `.eth`** (emoji, 1-char, >31) → no `.tru` equivalent; ignored.
- **Renewals during sunrise** → `renew` only touches existing leases; leave it
  open (no ENS gating needed).

---

## Test plan (mirror `azns/src/test.nr`)

- `sunrise_blocks_plain_register` — `register` reverts before `sunrise_deadline`.
- `sunrise_valid_voucher_registers` — happy path with a good attester sig.
- `sunrise_rejects_forged_voucher` — bad/other-key sig reverts.
- `sunrise_rejects_voucher_for_other_owner` — voucher bound to addr X, caller Y → revert.
- `sunrise_rejects_expired_voucher` — `now >= voucher_deadline` → revert.
- `sunrise_rejects_wrong_name` — voucher for `name_hash` A, register B → revert.
- `ga_opens_after_deadline` — plain `register` works at `deadline`, no voucher.
- `sunrise_still_charges_fee` — fee pulled exactly as normal registration.
- (Option B) `merkle_membership` + `ecdsa_binds_l1_owner` + non-membership if per-label.

---

## Rollout phases

1. **Spec sign-off** (this doc) + pick: global vs. per-label, fee policy, ENS-owner
   definition.
2. **Contract:** add `sunrise_deadline` + attester key, `register` guard,
   `register_with_ens_proof` (Option A). Full TXE test suite green.
3. **Attester service:** ownership-challenge + voucher signer + public log.
4. **dApp:** sunrise UI (connect L1 wallet → sign → fetch voucher → register).
5. **Snapshot + audit** of the added surface. Deploy with a concrete
   `sunrise_deadline`.
6. **(Later, optional)** Option B to drop the trusted attester.

---

## Open questions

- Window length? (ICANN sunrise is typically 30–60 days; you suggested a few weeks.)
- Global reservation or precise per-label (non-membership proofs)?
- Sunrise fee: normal, discounted, or minimum-multi-year?
- "ENS owner" = NameWrapper owner, registrant, or resolved address?
- Attester as single TEE key vs. k-of-n multisig at launch?
- Does an expired-but-in-cutoff `.eth` retain priority?

---

## Audit impact

Non-trivial. Adds: signature verification in a private circuit (Schnorr for A;
keccak+secp256k1 ECDSA + Merkle for B), a new privileged registration path, an
off-chain attester as a trust root (A), and voucher/replay semantics. All of it
must land **before** the external audit so it's in scope — it is not a
post-audit bolt-on.
