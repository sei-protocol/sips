**SIP-5: Post-Quantum Authorization for EVM Accounts**

| SIP Number | 5 |
| ----- | ----- |
| Title | Post-Quantum Authorization for EVM Accounts |
| Description | Adds post-quantum key binding, account level cutover, and a global classical signature cutoff for EVM accounts. |
| Author | [Maja Lie](mailto:maja@seinetwork.io) and [Benjamin Marsh](mailto:ben@seinetwork.io)  |
| Reviewer | [Philip Su](mailto:philip@seinetwork.io)           |
| Type | Standard (Core) |
| Created | 07/27/2026 |
| Status | Draft |
| Comments | https://github.com/sei-protocol/sips/discussions/15 |

## **Abstract**

Sei inherits the classical authentication assumptions of the EVM externally owned account model. 
Once an EOA has exposed its secp256k1 public key, a sufficiently capable quantum attacker could recover the private key and forge transactions from that account. 
This SIP introduces a staged migration mechanism that preserves existing EVM addresses, balances, nonces, code delegations, and contract relationships. 
An address can register a post-quantum verification key, begin using post-quantum transactions immediately or at a chosen block height, and rotate its registered key. 
A later governance set cutoff height disables classical EOA authorization across the network. After that cutoff, a bound EOA can originate transactions only with its registered post-quantum key. 
An unbound EOA cannot originate transactions, but its state remains intact and it remains a valid call and transfer target.

The initial algorithm profile is ML-DSA-44 as standardized in FIPS 204. 
The design retains an algorithm identifier so that later protocol upgrades can add or retire schemes without replacing the registry format.
New post-quantum accounts may be created and funded atomically after the cutoff through a sponsored transaction. 
A deterministic key derived registration path is also specified, but it is vulnerable to permanent dust griefing if its address is revealed before the binding is finalized. 
The cutoff therefore ends new classical EOA creation without ending EOA onboarding.


This mechanism is intended to be deployed well before a cryptographically relevant quantum computer exists. 
It provides a controlled migration path and an emergency cutoff that can be activated without a chain wide account reset.

## **Motivation**

A quantum migration performed only after a credible attack becomes public would be a race between account holders and attackers. The migration mechanism therefore needs to exist before the emergency.

The proposal has five objectives.

1. Existing EVM addresses and their state must survive the transition.
2. Account holders must be able to migrate before a global cutoff.
3. Validators must be able to verify post-quantum transactions through one address indexed lookup and one signature verification.
4. The protocol must preserve an onboarding path after classical EOAs are disabled.
5. The first deployment must be narrow enough to implement, audit, benchmark, and integrate into wallets before it is needed.

A registry is used because changing the EVM address width or replacing the account model would affect wallets, contracts, explorers, indexers, custody systems, and every 
address based integration. This SIP does not claim that a 20 byte registry backed account is the final post-quantum account model. It provides continuity while a longer term model is developed.

## **Threat Model**

This SIP considers an attacker that can recover a secp256k1 private key from an exposed public key within a time relevant to transaction authorization. 
The attacker may observe the public mempool, submit competing transactions, and exploit any account that still accepts a classical signature.
The attacker is assumed not to break ML-DSA-44, the collision and preimage properties of Keccak used by this SIP beyond the generic quantum bounds discussed below, or the consensus and governance mechanisms that activate the transition.
This is an EVM account authorization proposal and it does not by itself make Sei post-quantum secure. Consensus keys and other protocol keys require separate migration plans. 
We consider consensus out of scope for this work due to the ability to upgrade consensus keys at short notice and the ability to move to signatureless consensus protocols.
A global EOA cutoff is useful only if those systems remain secure or have also migrated.

## **Scope**

This SIP specifies:

- the post-quantum binding stored for an EVM address;
- the initial ML-DSA-44 algorithm profile;
- initial registration and key rotation;
- sponsored registration for accounts that do not yet hold funds;
- account level activation in `SOLO` or `DUAL` mode;
- a global cutoff for classical EOA authorization;
- a typed post-quantum transaction;
- post-quantum EIP-7702 authorization tuples;
- post-cutoff creation of post-quantum accounts through deterministic and assigned address paths;
- algorithm deprecation;
- gas and block resource accounting requirements;
- sponsored atomic creation and optional initial funding of new post-quantum accounts;
- RPC requirements needed by wallets and infrastructure.

This SIP does not specify:

- a permanent post-quantum address format wider than 20 bytes;
- recovery of an account that misses the cutoff;
- recovery after loss of the active post-quantum key;
- a zero-knowledge migration proof that hides a classical public key;
- migration of consensus, governance, bridge, oracle, or contract level authentication;
- signature aggregation or a final high throughput post-quantum transaction format;
- seizure, forfeiture, or sweeping of state held by an inoperable account.

Any recovery or rollback mechanism changes the authorization boundary and requires a separate SIP. In particular, this SIP does not permit governance to raise `H_Q` after the cutoff has taken effect and thereby reactivate classical authorization.

## **Specification**

The words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` are to be interpreted as normative requirements.
All integers are unsigned and use the canonical RLP integer encoding unless stated otherwise. All signatures and public keys MUST use canonical encodings. A decoder MUST reject trailing bytes, non-minimal integers, incorrect list lengths, incorrect fixed length fields, and any other non-canonical representation.

### **Constants**

| Name | Value | Meaning |
| --- | --- | --- |
| `PQ_AUTH_TX_TYPE` | `0x70` | EIP-2718 type byte for `PQAuthTx` |
| `SET_PQ_KEY_TX_TYPE` | `0x71` | EIP-2718 type byte for `SetPQKeyTx` |
| `ML_DSA_44_ID` | `0x01` | Algorithm identifier for FIPS 204 ML-DSA-44 |
| `SOLO` | `0x00` | Post-quantum only account activation |
| `DUAL` | `0x01` | Classical and post-quantum account activation |
| `NO_CUTOFF` | `2^64 - 1` | Sentinel meaning that no global cutoff is scheduled |
| `PQADDR_TAG` | ASCII `SEI_PQ_ADDR_V1__` | 16 byte post-quantum address domain tag |
| `PQCREATE_TAG` | ASCII `SEI_PQ_CREATE_V1` | 16-byte assigned-address derivation tag |
| `SIP5_ML_DSA_CONTEXT` | ASCII `SEI_SIP_5_V1` | FIPS 204 context for every ML-DSA operation in this SIP |
| `SIP5_AUTH_PREFIX` | Defined below | EIP-191-style prefix for non-transaction authorization messages |
| `AUTH_KIND_BIND` | `0x01` | Binding authorization message kind |
| `AUTH_KIND_POP` | `0x02` | Binding proof-of-possession message kind |
| `AUTH_KIND_7702` | `0x03` | EIP-7702 authorization message kind |
| `AUTH_KIND_CREATE_POP` | `0x04` | Atomic account creation proof-of-possession message kind |

A future change to any of these bytes requires a new transaction version and cannot reinterpret transactions already signed under this SIP.

`SIP5_AUTH_PREFIX` is the exact concatenation:

```text
0x19 ||
0x00 ||
0x0000000000000000000000000000000000000000 ||
ASCII("SEI_SIP_5_AUTH_V1")
```

Its first 22 bytes are an EIP-191 version `0x00` envelope with the all zero address as the native protocol validator identifier. 
It is followed by the exact 17 ASCII bytes `SEI_SIP_5_AUTH_V1`, one authorization kind byte, and the message specific RLP payload. 
No implementation may substitute a Unicode string, a length prefixed string, a hash of the string, or a null terminated string.

While this SIP is active, Sei MUST NOT assign `0x19` as an EIP-2718 transaction type. 
Reserving the envelope byte prevents a future typed transaction from turning a non-transaction ECDSA authorization into a transaction signature replay target.

### **Network Parameters**

The network stores the following consensus parameters.

| Parameter | Meaning |
| --- | --- |
| `H_reg` | First block height at which registration and `PQAuthTx` are valid |
| `H_Q` | First block height at which classical only EOA authorization is disabled |
| `AllowedPQAlgorithms` | Algorithms whose consensus verifiers are enabled |
| `AlgorithmDeprecations` | Map from a deprecated algorithm identifier to its deprecation height |
| `G_deprecate` | Minimum algorithm deprecation notice in blocks |
| `PQGasSchedule` | Consensus gas charges defined below |
| `MaxPQAuthBytesPerBlock` | Maximum post-quantum authentication bytes in one block |
| `G_cutoff_notice` | Minimum notice before scheduling or advancing the global cutoff |
| `MaxPQCreateProbes` | Maximum candidate addresses examined by one `CreatePQAccountTx` |


`H_req` MUST be less than or equal to `H_Q`. `H_Q` MAY initially equal `NO_CUTOFF`, allowing the migration mechanism to be deployed without scheduling the global cutoff.

Let `h` be the execution height of a governance update. Setting `H_Q` from `NO_CUTOFF` to a finite height, or replacing a finite `H_Q` with a lower height, is valid only if:

```text
new_H_Q >= h + G_cutoff_notice
```

Postponing a finite cutoff is permitted only before the current `H_Q` takes effect. 
Once a block at height `H_Q` has been finalized, increasing `H_Q` is outside the scope of this SIP. 
An emergency bypass of `G_cutoff_notice` is also outside the scope of this SIP.
A reduction in `G_cutoff_notice` itself MUST be announced under the old value and MUST NOT take effect earlier than `h + old_G_cutoff_notice`. 
A governance action MUST NOT reduce `G_cutoff_notice` and rely on the reduced value to schedule an earlier cutoff in the same proposal or activation batch.

The activation release MUST fix nonzero values for `G_cutoff_notice` and `G_deprecate`, and a positive value for `MaxPQCreateProbes`, before this SIP can move to Final.

An algorithm can appear in `AllowedPQAlgorithms` only after its exact encoding, verification procedure, test vectors, and gas schedule have been implemented by every consensus client. 
Governance can enable an implemented algorithm but governance alone cannot define a new verifier.

### **Initial ML-DSA-44 Profile**

The initial deployment MUST include exactly one algorithm identifier:

| Field | Value |
| --- | --- |
| `alg_id` | `ML_DSA_44_ID` |
| Public key length | 1,312 bytes |
| Signature length | 2,420 bytes |
| Verification | ML-DSA-44 verification under FIPS 204 |
| Security category | NIST category 2 |

The registry stores the canonical 1,312 byte public key. Expanded NTT representations are implementation caches and MUST NOT be consensus state. An implementation MAY cache an expanded key, but acceptance or rejection MUST depend only on the canonical key, message, context, and signature.

Verification MUST enforce every FIPS 204 encoding and norm check. In particular, implementations MUST reject malformed hints, duplicate or out-of-order hint indices, non-canonical coefficient encodings, and a `z` vector that violates the required norm bound. All clients MUST agree on every possible input, including malformed inputs.

For this SIP, `VerifyPQ(alg_id, pk, message, signature)` denotes the consensus verifier defined by the algorithm profile. For `ML_DSA_44_ID`, verification is exactly:

```text
ML-DSA.Verify(pk, M = message, signature, ctx = SIP5_ML_DSA_CONTEXT)
```

Signing is exactly `ML-DSA.Sign(sk, M = message, ctx = SIP5_ML_DSA_CONTEXT)`. 
The 32 byte Keccak digest supplied by this SIP is the FIPS 204 message `M`. Implementations MUST use pure ML-DSA. 
They MUST NOT invoke `HashML-DSA.Sign`, `HashML-DSA.Verify`, an external `mu` interface, or any library mode that pre-hashes `M` again outside the pure ML-DSA procedure.

Every later algorithm profile MUST define exactly how the 32 byte message is processed. 
The transaction type byte or the `SIP5_AUTH_PREFIX` and authorization kind byte provides purpose separation inside that message. 
The algorithm level context identifies SIP-5 and is not a second message kind taxonomy.

### **Binding State**

The protocol stores:

```text
PQBinding[address] = {
    alg_id: uint8,
    pk_pq: bytes,
    migration_nonce: uint64,
    activate_height: uint64,
    transition_mode: uint8
}
```

The absence of a binding is written `PQBinding[a] = null`.

`migration_nonce` orders binding updates. It is independent of the EVM account nonce. The EVM nonce orders transactions from a fee payer.
The migration nonce orders changes to one target address even when a third party pays for the registration transaction.

A binding is never deleted by ordinary execution. Algorithm deprecation also leaves the binding in state. This prevents an old registration authorization from becoming valid again and prevents a deprecated account from being treated as a fresh post-quantum derived account.

### **Post-Quantum Derived Addresses**

For a canonical post-quantum public key `pk_pq`, define:

```text
pq_address(alg_id, pk_pq) =
    keccak256(PQADDR_TAG || alg_id || pk_pq)[12:32]
```

An initially unbound address can use this derivation path only while the account is empty. 
For this purpose an account is empty when its balance is zero, its nonce is zero, its code is empty, and its storage root is empty.

The empty account requirement prevents post-cutoff self-registration from being used to claim a funded or previously used unbound EOA.
Registration can be sponsored, so a new post-quantum derived account SHOULD bind its key before receiving funds.

A protocol reserved address, native system address, or precompile address is never eligible for deterministic registration, even if its ordinary EVM account fields appear empty.
Removing it would allow a party that finds a 160 bit truncated key collision to claim a funded unbound EOA, including an EOA that missed `H_Q`. 
The generic quantum work factor for that targeted preimage problem is discussed in the Security Considerations section.

The same check creates a permanent dust griefing condition. 
If any party transfers a nonzero balance to the derived address before its binding is finalized, the derivation path becomes invalid. 
The intended key holder normally has no secp256k1 key that recovers to the derived address, so the classical initial binding path is unavailable. 
The received value cannot be spent, the balance cannot return to zero, and the address cannot later satisfy the derivation rule. 
After `H_Q`, it remains an unbound `INVALID` account. A one wei transfer is sufficient.

A wallet MUST treat a deterministic post-quantum derived address and its public key as secret until the binding transaction is irrevocably ordered. 
It MUST NOT display the address for deposits or broadcast the registration through a pending transaction path that exposes the address before ordering is fixed. 
If confidential or ordering protected submission is unavailable, the wallet MUST use `CreatePQAccountTx` instead.

The 16 byte tag separates this derivation from ordinary Ethereum address derivation at the input level. 
It does not turn the 20 byte output into a full 256 bit commitment. The security consequences of truncation are stated in the Security Considerations section.

### **`CreatePQAccountTx`**

`CreatePQAccountTx` is the safe default for onboarding a new post-quantum EOA. 
It assigns an unused address during execution, installs its binding, and optionally transfers initial value from the payer in the same state transition:

```text
CREATE_PQ_ACCOUNT_TX_TYPE || rlp([
    chain_id,
    payer_nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    payer,
    alg_id,
    pk_pq,
    creation_salt,
    initial_value,
    payer_alg_id,
    payer_classical_signature,
    payer_pq_signature,
    pop
])
```

`CreatePQAccountTx` is invalid before `H_reg`. `payer` MUST be exactly 20 bytes, `creation_salt` MUST be exactly 32 bytes, and `initial_value` is an unsigned 256 bit integer. 
The public key and signatures MUST have the exact canonical encodings required by their algorithms.

The payer signs:

```text
create_tx_message = keccak256(CREATE_PQ_ACCOUNT_TX_TYPE || rlp([
    chain_id,
    payer_nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    payer,
    alg_id,
    pk_pq,
    creation_salt,
    initial_value,
    payer_alg_id
]))
```

The payer is authenticated under its current authorization mode. A payer post-quantum signature is verified over `create_tx_message`. 
A payer classical signature is a recoverable secp256k1 signature over the same message and MUST recover `payer`. `payer_alg_id` follows the same rule as in `SetPQKeyTx`.

The new key proves possession over:

```text
create_pop_message = keccak256(
    SIP5_AUTH_PREFIX ||
    AUTH_KIND_CREATE_POP ||
    rlp([
        chain_id,
        payer,
        payer_nonce,
        alg_id,
        pk_pq,
        creation_salt,
        initial_value
    ])
)
```

`pop` MUST verify under `pk_pq` over `create_pop_message`.

`alg_id` MUST be in `AllowedPQAlgorithms` and must still accept new bindings. 
The payer nonce, balance, fee fields, gas limit, and authorization MUST satisfy the same rules as a `SetPQKeyTx`. 
Structural checks, payer state checks, and candidate state checks MUST occur before proof-of-possession verification.

For each integer `i` beginning at zero, let `uint64_be(i)` be the exact eight byte big endian encoding of `i`, and define:

```text
create_address(i) = keccak256(
    PQCREATE_TAG ||
    create_pop_message ||
    uint64_be(i)
)[12:32]
```

The created address is the first `create_address(i)` for `0 <= i < MaxPQCreateProbes` such that `PQBinding[create_address(i)] = null`, the EVM account is empty, and the address is not reserved for a native system function or precompile. If no candidate is eligible, the transaction is invalid and the payer may retry with a new `creation_salt`.

Candidate selection occurs before the binding and value transfer. Dust sent to one predicted candidate therefore causes the transaction to select the next eligible candidate rather than permanently bricking the new account. Dusting every candidate can delay one creation attempt, but it cannot claim the key or prevent a retry with a new salt.

On success, the protocol atomically:

1. authenticates the payer and proof of possession;
2. increments the payer's EVM nonce;
3. creates a binding with `migration_nonce = 0`, `transition_mode = SOLO`, and `activate_height = min(current_height, H_Q)`;
4. transfers `initial_value` from the payer to the created address;
5. charges the payer for fees, every candidate probe, verification, persistent state, and value.

The payer balance MUST cover `initial_value` in addition to the transaction's maximum gas liability. 
The created account is immediately `PQ_ONLY`. The transaction performs no EVM call and creates no code or storage. 
RPC receipts for this type MUST expose the selected address as `createdPQAddress`.

### **`PQAuthTx`**

`PQAuthTx` is an EIP-2718 typed transaction:

```text
PQ_AUTH_TX_TYPE || rlp([
    chain_id,
    nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    to,
    value,
    data,
    access_list,
    authorization_list,
    sender,
    alg_id,
    classical_signature,
    pq_signature
])
```

`PQAuthTx` is invalid before `H_reg`.

The ordinary execution fields have the same meaning as in an EIP-1559 transaction. `sender` MUST be exactly 20 bytes. 
`alg_id` MUST be a one byte algorithm identifier. `classical_signature` is either empty or a 65 byte value encoded as `y_parity || r || s`. `pq_signature` has the exact length required by `alg_id`.

The signing message is:

```text
tx_message = keccak256(PQ_AUTH_TX_TYPE || rlp([
    chain_id,
    nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    to,
    value,
    data,
    access_list,
    authorization_list,
    sender,
    alg_id
]))
```

The post-quantum signature is verified over the 32 byte `tx_message` using `PQ_TX_CONTEXT`. 
When a classical signature is required, it is a recoverable secp256k1 signature over the same `tx_message`. 
The recovered address MUST equal `sender`, and the EIP's low `s` rule applies.

`alg_id` MUST equal `PQBinding[sender].alg_id`. The verifier MUST reject an unsupported or mismatched identifier before performing post-quantum verification.

The receipt format is the existing EIP-2718 receipt format. The transaction origin and top level message sender are the explicit `sender` after successful authentication.

### **Authorization Modes**

Let `B = PQBinding[a]`. An algorithm is `disabled` for a binding only after its deprecation deadline has passed. During the deprecation notice window it remains enabled for existing bindings.

The authorization mode table is priority ordered and the first matching row is authoritative. 
In particular, the global cutoff and disabled algorithm rows are evaluated before either comparison with `activate_height`.

The authorization mode for address `a` at height `h` is:

| Condition | Mode |
| --- | --- |
| `B` exists and its algorithm is disabled | `ALGORITHM_DISABLED` |
| `B = null` and `h < H_Q` | `LEGACY` |
| `B = null` and `h >= H_Q` | `INVALID` |
| `B` exists, its algorithm is enabled, and `h >= H_Q` | `PQ_ONLY` |
| `B` exists, `h < B.activate_height`, and `B.transition_mode = DUAL` | `LEGACY_OR_DUAL` |
| `B` exists, `h < B.activate_height`, and `B.transition_mode = SOLO` | `LEGACY_OR_PQ` |
| `B` exists, `B.activate_height <= h < H_Q`, and `B.transition_mode = DUAL` | `DUAL_ONLY` |
| `B` exists, `B.activate_height <= h < H_Q`, and `B.transition_mode = SOLO` | `PQ_ONLY` |

The permitted ordinary transaction authorizations are:

| Mode | Legacy EOA transaction | `PQAuthTx` |
| --- | --- | --- |
| `LEGACY` | Valid under existing rules | Invalid |
| `LEGACY_OR_DUAL` | Valid | Valid only with valid classical and PQ signatures |
| `LEGACY_OR_PQ` | Valid | Valid with a valid PQ signature. A non-empty classical signature MUST also be valid |
| `DUAL_ONLY` | Invalid | Valid only with valid classical and PQ signatures |
| `PQ_ONLY` | Invalid | Valid only with a valid PQ signature and an empty classical signature |
| `INVALID` | Invalid | Invalid |
| `ALGORITHM_DISABLED` | Invalid | Invalid |

After `H_Q`, no ordinary transaction validation path invokes classical recovery for the sender. `LEGACY_OR_DUAL` is not a two factor security mode and before activation, a classical signature alone can originate a legacy transaction and can authorize replacement of the binding. A registered post-quantum key does not protect an account from compromise of its classical key until the account enters `DUAL_ONLY` or `PQ_ONLY`.

`INVALID` and `ALGORITHM_DISABLED` apply only to origination and new authorization. 
The address remains a valid recipient of value transfers and remains a valid target for `CALL`, `CALLCODE`, `DELEGATECALL`, and `STATICCALL`. 
Balance, nonce, storage, and code are preserved.

### **`SetPQKeyTx`**

`SetPQKeyTx` writes or replaces one binding and allows a separate account to pay the transaction fee:

```text
SET_PQ_KEY_TX_TYPE || rlp([
    chain_id,
    payer_nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    payer,
    address,
    alg_id,
    pk_pq,
    migration_nonce,
    activate_height,
    transition_mode,
    payer_alg_id,
    payer_classical_signature,
    payer_pq_signature,
    auth_classical_signature,
    auth_pq_signature,
    pop
])
```

`payer` and `address` MUST each be exactly 20 bytes. The payer pays gas and its EVM nonce is incremented. 
`address` is the account whose binding is changed. The two addresses MAY be equal.

The payer signs:

```text
set_key_message = keccak256(SET_PQ_KEY_TX_TYPE || rlp([
    chain_id,
    payer_nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    payer,
    address,
    alg_id,
    pk_pq,
    migration_nonce,
    activate_height,
    transition_mode,
    payer_alg_id
]))
```

The payer is authenticated under the mode that applies to `payer` at the current height, using the same permitted signature combinations as an ordinary transaction. 
A payer post-quantum signature is verified over `set_key_message` using `PQ_TX_CONTEXT`. 
A payer classical signature is a recoverable secp256k1 signature over the same message and MUST recover `payer`.

`payer_alg_id` MUST equal zero when `PQBinding[payer] = null` and MUST otherwise equal the algorithm identifier in the payer's binding.

The permitted payer authorizations for both `SetPQKeyTx` and `CreatePQAccountTx` are:

| Payer mode | Required payer authorization |
| --- | --- |
| `LEGACY` | Classical |
| `LEGACY_OR_DUAL` | Classical, or classical and current PQ |
| `LEGACY_OR_PQ` | Classical, current PQ, or both |
| `DUAL_ONLY` | Classical and current PQ |
| `PQ_ONLY` | Current PQ |
| `INVALID` | No transaction permitted |
| `ALGORITHM_DISABLED` | No transaction permitted |

Every non-empty payer signature MUST verify. A post-quantum payer signature is invalid if no binding exists.

The target binding fields are authenticated through:

```text
binding_message = keccak256(
    SIP5_AUTH_PREFIX ||
    AUTH_KIND_BIND ||
    rlp([
        chain_id,
        address,
        alg_id,
        pk_pq,
        migration_nonce,
        activate_height,
        transition_mode
    ])
)
```

The new key proves possession through:

```text
pop_message = keccak256(
    SIP5_AUTH_PREFIX ||
    AUTH_KIND_POP ||
    rlp([
        chain_id,
        address,
        alg_id,
        pk_pq,
        migration_nonce,
        activate_height,
        transition_mode
    ])
)
```
`auth_classical_signature` is a recoverable secp256k1 signature over `binding_message` and MUST recover `address`. 
`auth_pq_signature` is verified over `binding_message` using the currently registered algorithm profile. 
`pop` is verified over `pop_message` using the proposed new algorithm profile and key. 
Every non-empty authorization field MUST verify, even when the current mode would permit authorization without that field.

If `payer = address`, the payer authentication MAY also serve as the target authorization. 
In that case the two target authorization fields MUST be empty. 
The payer signature covers every binding field through `set_key_message`, and the independent proof of possession remains required.

If `payer != address`, the target authorization fields MUST contain the authorization required by the target's current mode. 
This permits a relayer, exchange, custodian, application, or other account to pay for migration without gaining control of the binding.

### **Binding Validation**

A `SetPQKeyTx` is valid only if every applicable condition below holds.

1. The current height is at least `H_reg`.
2. `chain_id` equals the current chain ID.
3. `alg_id` is in `AllowedPQAlgorithms` and is not under deprecation for new bindings.
4. `pk_pq` and every signature have the exact canonical encoding and length required by their algorithms.
5. The payer is validly authenticated under its current mode.
6. The payer nonce, balance, fee fields, and gas limit satisfy the ordinary transaction rules.
7. `transition_mode` is `SOLO` or `DUAL`.
8. The activation height rules below hold.
9. The migration nonce rules below hold.
10. The target authorization rules below hold.
11. `pop` is a valid signature under the proposed `pk_pq`.

Cheap structural, state, nonce, balance, and fee checks MUST occur before post-quantum verification.

#### Activation Height

For any initial binding before `H_Q`:

```text
current_height <= activate_height <= H_Q
```

Omitting `activate_height` sets it to the current value of `H_Q`. Omitting `transition_mode` sets it to `SOLO`.

If `transition_mode = DUAL` before `H_Q`, `activate_height` MUST be strictly less than `H_Q`. Otherwise the dual only interval would be empty.

Consequently, `activate_height` MUST be explicitly supplied whenever `transition_mode = DUAL`. 
Omitting it would set it equal to `H_Q` and make the transaction invalid. This remains true while `H_Q = NO_CUTOFF`.
For an account whose secp256k1 public key has previously been exposed, a wallet MUST default to `activate_height = current_height`. 
A future activation height leaves the account fully exposed to a classical key compromise until activation. The user MAY override this default only after an explicit warning.

For an initial post-quantum derived registration at or after `H_Q`, `activate_height` MUST equal `H_Q` and `transition_mode` MUST equal `SOLO`. 
The account is immediately in `PQ_ONLY` mode because the global cutoff has already taken effect.

For a replacement binding:

```text
new_activate_height <= min(old_activate_height, H_Q)
```

A replacement therefore cannot postpone an account's cutover or reactivate classical-only authorization. 
The account MAY change between `SOLO` and `DUAL`, subject to authorization under its current mode.

At or after `H_Q`, a replacement MUST use `SOLO`. The transition field no longer changes authorization after the global cutoff.

#### Migration Nonce

If `PQBinding[address] = null`, `migration_nonce` MUST equal zero.

If a binding exists, including a binding whose algorithm is in its deprecation notice window, `migration_nonce` MUST equal the stored migration nonce plus one.

An `ALGORITHM_DISABLED` binding is not null and cannot be replaced under this SIP because its old key is no longer accepted. The deprecation window is the opportunity to rotate.

#### Target Authorization

For an initial binding before `H_Q`, one of the following is required:

1. A recoverable secp256k1 signature over `binding_message` whose recovered address equals `address`.
2. The post-quantum derived registration path below.

For an existing enabled binding, the authorization signatures MUST satisfy the target's current mode:

| Target mode | Required target authorization |
| --- | --- |
| `LEGACY_OR_DUAL` | Classical, or classical and current PQ |
| `LEGACY_OR_PQ` | Classical, current PQ, or both |
| `DUAL_ONLY` | Classical and current PQ |
| `PQ_ONLY` | Current PQ |
| `ALGORITHM_DISABLED` | No update permitted |

The authorization signature under the current key and the proof of possession under the new key are separate requirements.
A rotation cannot install a key for which possession has not been proved.

#### Post-Quantum Derived Registration

The derivation path is valid when all of the following hold:

1. `PQBinding[address] = null`.
2. The account at `address` is empty.
3. The address is not reserved for a native system function or precompile.
4. `address = pq_address(alg_id, pk_pq)`.
5. `pop` verifies under `pk_pq`.
6. The remaining binding and transaction rules hold.

The derivation equation and proof of possession replace target authorization. They do not replace payer authorization.

This path is available from `H_reg` onward, including after `H_Q`. 
It is the deterministic native onboarding path. `CreatePQAccountTx` is the default path when confidential or ordering-protected submission is unavailable.

### **Key Rotation**

A key rotation is a `SetPQKeyTx` for an address with an existing enabled binding. It is authorized under the current binding and installs a new binding only after the new key passes proof of possession.

The transaction MAY change `alg_id`, `pk_pq`, `transition_mode`, and `activate_height`, subject to the rules above. It MUST increment `migration_nonce` by exactly one.

Rotation is atomic. The old binding remains active if the transaction fails. 
A transaction already signed by the old key but executed after a successful rotation is invalid because verification uses the binding in state at execution time.

### **EIP-7702**

An EIP-7702 authorization tuple is an account authorization action. The authority's mode is evaluated independently from the outer transaction sender's mode.

The existing six element EIP-7702 tuple remains the classical tuple:

```text
[chain_id, delegate, nonce, y_parity, r, s]
```

This tuple is permitted only when the authority's mode permits a classical only authorization.

This SIP adds two extended tuple formats:

```text
pq_tuple = [
    1,
    chain_id,
    delegate,
    nonce,
    authority,
    alg_id,
    pq_signature
]

dual_tuple = [
    2,
    chain_id,
    delegate,
    nonce,
    authority,
    alg_id,
    classical_signature,
    pq_signature
]
```

For either extended tuple:

```text
authorization_message = keccak256(
    SIP5_AUTH_PREFIX ||
    AUTH_KIND_7702 ||
    rlp([
        auth_type,
        chain_id,
        delegate,
        nonce,
        authority,
        alg_id
    ])
)
```

The post-quantum signature is verified over `authorization_message` using the authority's current algorithm profile and binding. 
In a dual tuple, the classical signature MUST recover `authority` from the same message. `alg_id` MUST match the current binding.

For an extended tuple, `chain_id` MUST equal the current chain ID. 
The cross-chain value zero permitted by the original EIP-7702 classical tuple is not permitted in an extended tuple.

The permitted tuple forms are:

| Authority mode | Permitted EIP-7702 tuple |
| --- | --- |
| `LEGACY` | Existing classical tuple |
| `LEGACY_OR_DUAL` | Existing classical tuple or extended dual tuple |
| `LEGACY_OR_PQ` | Existing classical tuple, extended PQ tuple, or extended dual tuple |
| `DUAL_ONLY` | Extended dual tuple |
| `PQ_ONLY` | Extended PQ tuple |
| `INVALID` | None |
| `ALGORITHM_DISABLED` | None |

`PQAuthTx.authorization_list` can carry any tuple permitted by the authority mode table. 
The legacy EIP-7702 transaction type continues to carry only the existing classical tuple. 
If an authorization list is non-empty, the `to` field MUST not be null and processing follows EIP-7702 ordering, nonce, code delegation, gas, and failure semantics except for the authentication changes above.

The outer sender of an EIP-7702 transaction must independently satisfy its own mode. After `H_Q`, the legacy EIP-7702 transaction type cannot be used by an EOA sender.
A `PQAuthTx` with a non-empty authorization list provides the post-quantum outer transaction path.

The existing classical tuple retains the EIP-7702 rule that `chain_id = 0` authorizes across chains. 
This remains true when a classical tuple appears inside `PQAuthTx.authorization_list` and while the authority is in a mode that permits classical authorization. 
Wallets MUST describe that scope explicitly and SHOULD use the current nonzero chain ID unless cross-chain authorization is the user's stated intent. 
Extended PQ and dual tuples never permit zero.

An existing delegation is not revoked when its authority changes mode or becomes `INVALID`. 
The delegated code continues to run on inbound calls. The mode controls transaction origination and the creation of new authorization tuples, not code execution.

### **Algorithm Deprecation**

Governance MAY announce an algorithm deprecation. The announcement records `AlgorithmDeprecations[alg_id] = H_dep`, and:

```text
H_dep >= announcement_height + G_deprecate
```

The announcement is valid only if at least one different algorithm is already in `AllowedPQAlgorithms`, accepts new bindings at the announcement height, and is not scheduled for deprecation at or before `H_dep`. 
Governance MUST NOT deprecate the last live onboarding algorithm. A replacement verifier, encoding, gas schedule, and cross-client test suite must be active before the old algorithm's notice period begins.

From the announcement height:

- new bindings using the algorithm are invalid;
- existing bindings remain fully usable;
- affected accounts may rotate to another allowed algorithm.

At `H_dep`, an affected address enters `ALGORITHM_DISABLED`. The binding remains in state, but it cannot authorize transactions or rotation under this SIP.

The verifier MAY be removed from `AllowedPQAlgorithms` at `H_dep`. It MUST NOT be removed earlier while any binding uses it. 
Any update to `AllowedPQAlgorithms` that would leave no algorithm accepting new bindings is invalid.

Deprecation MUST NOT treat the binding as absent and MUST NOT reactivate classical authorization. 
Automatically falling back to ECDSA would turn an algorithm safety action into an account takeover path.

Governance SHOULD reinstate an algorithm only through a coordinated client release and SHOULD NOT shorten an announced deprecation window. 
Any recovery path for an account that failed to rotate requires a separate SIP.

### **RPC Requirements**

Every RPC implementation that exposes EVM state MUST expose equivalent methods for:

```text
sei_getPQBinding(address, block_tag)
sei_getPQAuthorizationMode(address, block_tag)
sei_getPQParameters(block_tag)
```

`sei_getPQBinding` returns `null` or the complete binding fields, the algorithm's deprecation state, and any announced deprecation height. 
`sei_getPQAuthorizationMode` returns the mode obtained from the consensus mode table.
`sei_getPQParameters` returns `H_reg`, `H_Q`, `G_cutoff_notice`, `G_deprecate`, `AllowedPQAlgorithms`, `AlgorithmDeprecations`, `MaxPQAuthBytesPerBlock`, and `MaxPQCreateProbes`. The ordinary transaction receipt RPC response for a successful `CreatePQAccountTx` MUST include the selected 20 byte `createdPQAddress`.

RPC responses are informational. Transaction validity is determined from consensus state at the execution block.

### **Gas Accounting**

Post-quantum authentication consumes CPU, bandwidth, block storage, and persistent state. 
Standard transaction calldata accounting does not automatically charge for bytes in a typed transaction envelope. 
Registry access during transaction validation is also not an EVM `SLOAD`. This SIP therefore requires explicit consensus charges.

`PQGasSchedule` contains at least:

| Gas component | Purpose |
| --- | --- |
| `G_pq_verify[alg_id]` | One post-quantum verification |
| `G_additional_ecdsa` | One additional classical recovery in a dual authorization |
| `G_binding_read` | One registry lookup in transaction validation |
| `G_auth_byte` | Each byte in the canonical RLP encoding of authentication envelope fields, including their RLP prefixes |
| `G_binding_create[alg_id]` | Creation of a persistent binding |
| `G_binding_replace[alg_id]` | Replacement of a persistent binding |
| `G_create_probe` | One account and binding lookup during assigned address selection |

The activation release MUST fix every value and publish reproducible benchmarks across all supported client implementations and validator architectures.

The initial recommended draft values are:

| Component | Draft value |
| --- | --- |
| `G_pq_verify[ML_DSA_44_ID]` | 50,000 gas |
| `G_additional_ecdsa` | 3,000 gas |
| `G_binding_read` | 2,100 gas |
| `G_auth_byte` | 16 gas |

The binding create and replace charges depend on Sei's native state cost schedule and MUST be fixed before this SIP can move to Final.

For an ML-DSA-44 `PQAuthTx`, the RLP encodings of the sender, algorithm identifier, empty classical signature field, and 2,420 byte post-quantum signature occupy 2,446 bytes. 
Under the draft values, a simple post-quantum transfer with no access list has an illustrative intrinsic floor of:

```text
21,000
+ 2,446 * 16
+ 50,000
+ 2,100
= 112,236 gas
```

A 65 byte classical signature has a 67 byte RLP encoding and replaces the one byte empty field. 
A dual signed transaction therefore adds 66 encoded bytes and one classical recovery:

```text
112,236
+ 66 * 16
+ 3,000
= 116,292 gas
```

These values exclude execution data, access list charges, EIP-7702 tuples, and contract execution. They are not derived by pretending that the binding lookup is an `SLOAD`.

A `SetPQKeyTx` additionally pays for the public key, proof of possession, any current key authorization, every required verification, and the binding write. 
If `payer = address` and one signature serves both payer and target authorization, the verifier and gas schedule MUST count it once.

A `CreatePQAccountTx` additionally pays `G_create_probe` for every candidate examined, including an occupied candidate, plus proof-of-possession verification, binding creation, authentication bytes, and any payer authorization.

An ML-DSA-44 binding contains at least 1,330 raw bytes before trie or database overhead in the form of a 1,312 byte public key and 18 bytes of fixed metadata. 
One million bindings therefore require at least 1.33 GB of raw current state payload. 
Because ordinary execution never deletes a binding, current registry size is monotonic in the number of addresses that have ever registered.

Before a finite `H_Q` is scheduled, the activation release MUST publish projected registry size under expected and worst case migration, measured database amplification, snapshot and state sync costs, and archival growth. 
The gas schedule must recover the persistent cost under Sei's state pricing policy. 
An active public key cannot simply be pruned because transaction verification requires it.
Any design that moves keys to an authenticated external table, supplies key witnesses with transactions, or verifies key use inside a validity proof changes the state access model and requires a separate SIP.

### **Block Resource Limit**

Gas alone is not a sufficient network bandwidth limit unless the gas schedule is calibrated to the full transaction envelope. 
Every block MUST also enforce `MaxPQAuthBytesPerBlock`, measured over the canonical encoded sender, algorithm, public key, and signature fields introduced by this SIP, including their RLP prefixes.

ML-DSA-44 adds 2,420 signature bytes per post-quantum transaction. 
At 200,000 transactions per second, signatures alone would require approximately 484 MB per second before transaction encoding, propagation overhead, erasure coding, or replication. We therefore note the need for a different mechanism to maintain throughput.

Governance MUST set `MaxPQAuthBytesPerBlock` from measured propagation and storage limits. A finite `H_Q` MUST NOT be scheduled on the assumption that a future non-interactive aggregation method for ML-DSA or another lattice signature will become available. 
No such method is part of this SIP.

## **Rationale**

### **Why Preserve Existing Addresses**

Balances are only one part of an account's identity. 
Addresses appear in token approvals, access control lists, vesting contracts, staking positions, bridges, exchange records, custody policies, and off-chain databases. 
A forced move to a new address would require every dependent system to migrate correctly during the same emergency.

A binding preserves that identity while replacing the authorization key.

### **Why Use a Registry**

The registry makes verification direct:

1. parse the explicit sender;
2. load the binding;
3. determine the mode;
4. verify the required signature or signatures;
5. execute the transaction.

Storing the key in a dedicated registry also avoids overloading the EVM code field. 
This matters for accounts that already use, or may later use, EIP-7702 delegation. 
Draft native key delegation proposals that place the key in account code do not provide the staged global cutoff, mode transition, or independent code delegation semantics required here.

### **Why Use an Explicit Sender**

A recoverable ECDSA signature identifies its sender and ML-DSA does not. 
The transaction must state the sender so the verifier knows which registered key to load. The sender is included in every signing message, so it cannot be substituted by a relayer.

### **Why Allow `SOLO` and `DUAL`**

`SOLO` allows an account to stop relying on ECDSA and operate with its post-quantum key alone. It is the simpler mode and the default.

`DUAL` requires both keys only after the account's activation height and before the global cutoff. 
In `DUAL_ONLY`, it protects against compromise of either key in isolation, but it also inherits the availability risk of both keys. 
Before activation, `LEGACY_OR_DUAL` still permits a classical only transaction and a classically authorized rotation. 
It provides compatibility and testing, not two factor security. At `H_Q`, the classical factor is removed because the global policy no longer treats it as trustworthy.

The activation height monotonicity rule makes an account level cutover irreversible under ordinary key rotation. 
A compromised key cannot rotate the binding and postpone activation to re-enable classical only transactions.
For a used EOA whose secp256k1 public key is already visible on chain, delayed activation leaves the dominant quantum risk unchanged. 
Immediate `SOLO` or immediate `DUAL_ONLY` should be the normal migration choice. 
Delayed activation is an operational compatibility option and must not be presented as protection.

### **Why Permit Sponsored Registration**

A new post-quantum derived account should bind its key before receiving funds. 
Requiring the target account to pay for its own registration would force it to be funded while still unbound. 
A separate payer removes that interval and also supports exchanges, custodians, applications, and migration services that pay registration costs for users.

The payer controls fees and inclusion. The target controls the binding. Neither role implies the other.

### **Why Support Two Post-Quantum Onboarding Paths**

The global cutoff closes classical EOA creation. It should not require all future users to rely on pre-registered stockpiles of classical addresses.
The deterministic derivation path permits a fresh empty address to prove that its address and post-quantum key correspond. 
It is useful when an application needs the address to be derived directly from the key, but it has the irreversible dust griefing tradeoff described above.

`CreatePQAccountTx` is the safer general path. It selects an unused address during execution, skips occupied candidates, installs the binding, and can fund the account atomically. 
The address may change if a candidate is occupied, so applications learn the final address from the receipt rather than treating a precomputed candidate as final.
Once either binding exists, later changes require authorization under the registered key. 
Contract wallets remain an independent onboarding path.

### **Why ML-DSA-44**

ML-DSA is standardized in FIPS 204. ML-DSA-44 has a 1,312 byte public key, a 2,420 byte signature, and mature constant time implementations. 
It is a conservative first algorithm for a migration feature that must work in browser wallets, mobile wallets, hardware devices, HSMs, custody systems, and validator clients.

FN-DSA offers substantially smaller signatures and public keys, directly reducing the dominant bandwidth cost in this SIP.
NIST had not published a final FIPS 206 when this SIP was created, and the public FIPS 206 status material described its encodings and implementation rules as provisional. 
Its signing implementation is also substantially more delicate because of floating point sampling and side-channel requirements.
Once an exact FIPS 206 standard is available, a later client release can define a new `alg_id`, canonical key and signature encodings, consensus verifier, gas schedule, test vectors, and wallet profile. 
The variable length registry and per-algorithm gas maps in this SIP are designed to accommodate that addition without reinterpreting `ML_DSA_44_ID`.

### **Why Signature Aggregation Is Not Assumed**
This SIP does not assume that independent lattice signatures will gain a practical, non-interactive aggregation operation. 
A proof can attest that many ordinary signatures verified against a committed registry root without algebraically combining those signatures. 
It may reduce validator verification work and, if combined with an appropriate data availability design, reduce what consensus nodes must download. 
It also introduces a prover and proof system dependency. That tradeoff must be evaluated directly and as such is out of scope for now.

### **Why No Hidden Key Migration Proof**

An ECDSA transaction reveals the classical public key. 
A zero-knowledge proof of control could migrate an unexposed address without that disclosure, but the proof system, quantum soundness, statement encoding, verifier, trusted setup assumptions, and gas cost are all consensus critical.

An unspecified proof placeholder would not be implementable and could create a second emergency attack surface. 
This SIP therefore relies on migration before a practical attack and leaves a hidden key authorization path to a separate SIP.

### **Why Algorithm Deprecation Locks Instead of Downgrading**

If a deprecated binding were treated as absent before `H_Q`, the account could silently return to classical authorization. 
That is unsafe for an account that deliberately completed its post-quantum cutover.

The deprecation window provides time to rotate. After the deadline, locking is safer than silently choosing a weaker key. 
Any exceptional recovery policy must be explicit and separately reviewed.

## **Backwards Compatibility**

Before an account's activation height and before `H_Q`, ordinary EOA transactions remain valid wherever the mode table permits them. 
Existing addresses, balances, nonces, storage, contract relationships, and EIP-7702 delegations are not rewritten.

This SIP is intentionally not backward compatible with indefinite classical EOA origination. An unbound EOA cannot originate transactions after `H_Q`. 
Applications that assume every 20 byte address can always initiate a transaction must handle `INVALID` and `ALGORITHM_DISABLED` accounts.

Contract accounts and account abstraction systems are not directly changed. 
They are not automatically protected either. A contract wallet that verifies ECDSA internally remains vulnerable until its own authorization logic is upgraded.

## **Operational Requirements**

Activation should occur in three stages.

### **Stage 1: Registration Support**

At `H_reg`, clients enable the registry, all three typed transactions, RPC methods, gas accounting, block byte accounting, and test vectors. `H_Q` may remain `NO_CUTOFF`.

Wallets should support:

- ML-DSA-44 key generation and protected backup;
- proof-of-possession generation;
- sponsored and self-paid registration;
- mode and activation height selection;
- `PQAuthTx` signing;
- sponsored `CreatePQAccountTx` with atomic initial funding;
- confidential or ordering protected submission for deterministic derived address registration;
- binding and deprecation queries;
- key rotation;
- clear warnings that early activation and missed cutoffs may be irreversible.

Wallets MUST default a used EOA to immediate activation. They MUST default new user onboarding to `CreatePQAccountTx` unless the deterministic derived address is required and can remain undisclosed until ordering is fixed.

### **Stage 2: Migration**

Infrastructure providers, exchanges, custodians, hardware vendors, and applications integrate the new formats. The network monitors:

- the percentage and value of EOA state protected by bindings;
- verification latency by client and hardware class;
- signature byte utilization per block;
- registry growth;
- malformed transaction and peer level abuse rates;
- wallet and custody support.

At least one production sponsor or protocol fee grant path for users with no existing valid account MUST be operational before a finite `H_Q` is scheduled.

### **Stage 3: Global Cutoff**

Governance schedules a finite `H_Q` only after the mechanism has been deployed, audited, exercised on test networks, and supported by critical infrastructure. 
The on-chain update MUST satisfy `G_cutoff_notice`, the operational process cannot waive that consensus rule. 
Nodes MUST evict legacy transactions that cannot remain valid at the cutoff. 
Wallets and RPC providers MUST report the impending mode change throughout the notice period.

Mode evaluation is by execution block height. 
Reorganizations across `H_Q`, an account activation height, or an algorithm deprecation height are handled by ordinary state re-execution.

## **Mempool Policy**

The following rules are non-consensus policy recommendations.

- Perform type, length, canonical encoding, chain ID, mode, binding, nonce, balance, fee, and block byte limit checks before cryptographic verification.
- Rate limit `PQAuthTx` and `SetPQKeyTx` per peer and per claimed sender or target.
- Cache successful and failed verification results by transaction hash.
- Retain at most one replacement candidate for a given payer and payer nonce, following ordinary fee bump rules.
- Limit the number of queued binding updates for one target.
- Revalidate queued transactions when a binding rotates, an activation height is crossed, `H_Q` is crossed, or an algorithm deprecation state changes.
- Do not rely on the offered fee alone as a DoS defense. An invalid signature cannot be charged.

## **Test Requirements**

Before activation, the client repository MUST contain shared cross-client vectors covering at least:

- valid and invalid ML-DSA-44 signatures;
- every malformed public key, signature, hint, coefficient, and norm bound case required by FIPS 204;
- every authorization mode immediately before, at, and after its boundary height;
- sender mismatch and algorithm mismatch;
- chain replay, nonce replay, and migration nonce replay;
- initial classical migration;
- initial post-quantum derived registration;
- sponsored registration;
- same account payer and target registration;
- rotation within and across algorithms;
- attempted activation height postponement;
- proof-of-possession failure;
- EIP-7702 classical, PQ, and dual tuples in every mode;
- rotation invalidating a queued old key transaction;
- deprecation announcement, grace window, and deadline;
- rejection of a deprecated binding without ECDSA fallback;
- block authentication byte limit enforcement;
- re-execution across relevant height boundaries;
- rejection of a derived registration after a one wei pre-funding transfer;
- assigned address creation, occupied or reserved candidate skipping, atomic initial funding, probe exhaustion, and retry with a new salt;
- post-cutoff onboarding through an existing post-quantum sponsor;
- invalid `DUAL` registration with omitted `activate_height`;
- classical only rotation in `LEGACY_OR_DUAL` and two signature rotation in `DUAL_ONLY`;
- rejection of HashML-DSA and external `mu` processing in place of pure ML-DSA;
- exact `SIP5_AUTH_PREFIX` and authorization kind encodings;
- rejection of `0x19` as an EIP-2718 transaction type;
- acceptance of classical EIP-7702 `chain_id = 0` where inherited rules permit it and rejection of zero in every extended tuple;
- rejection of a deprecation that would leave no live onboarding algorithm;
- initial and advanced cutoff scheduling at exactly, below, and above `G_cutoff_notice`;
- delayed effectiveness of a `G_cutoff_notice` reduction;
- priority ordered mode evaluation when `H_Q` is reduced below a stored `activate_height`;
- candidate probe and persistent binding gas accounting.

Consensus fuzzing MUST compare all supported clients on malformed RLP, malformed cryptographic encodings, and boundary values.

## **Security Considerations**

### **Migration Must Precede the Attack**

This SIP cannot securely rescue an account after an attacker can already forge its classical signature. 
A last minute registration transaction can be copied, frontrun, censored, or replaced by an attacker who recovers the classical key from the mempool disclosure.

The safe strategy is to deploy and use the mechanism before a practical attack. 
Accounts whose public keys have already been exposed should migrate first and should use an immediate activation height when operationally possible.

### **Public Key Exposure**

An ordinary classical registration authorization reveals enough information to recover the secp256k1 public key. 
Registration simultaneously installs the post-quantum key, but it is not safe against an attacker that can break ECDSA within the transactio -propagation window.

The hidden key migration path discussed in the Rationale would reduce this exposure but requires a separate complete proof system specification.

### **Replay and Domain Separation**

Every signed message includes the chain ID, target or sender address, algorithm identifier, and all fields that affect authorization. 
Transactions include the ordinary EVM nonce. Binding updates include a monotonic migration nonce.

Transaction signatures are separated by their EIP-2718 type bytes. 
Non-transaction authorizations use the pinned `SIP5_AUTH_PREFIX` followed by a pinned authorization kind byte. 
The EIP-191 style leading bytes keep a classical authorization digest out of the legacy transaction encoding, while the exact prefix and kind prevent one SIP-5 authorization purpose from being reinterpreted as another.

All ML-DSA operations use one pinned FIPS 204 context and sign the already domain separated 32 byte digest as the pure ML-DSA message. 
A client that substitutes a different ASCII string, HashML-DSA, or an external `mu` interface risks a consensus split.

### **Proof of Possession**

Without proof of possession, an address could be bound to an unusable key or to a key controlled by another party. 
Every initial binding and rotation therefore requires a signature under the proposed new key.

### **Rotation and Downgrade Resistance**

Before activation, a mode that permits classical authorization also permits a classical attacker to attempt a binding replacement. 
`LEGACY_OR_DUAL` therefore provides no protection against isolated compromise of the classical key. Used EOAs should activate immediately.

After activation, replacement cannot postpone `activate_height`. `PQ_ONLY` rotation requires the current post-quantum key. 
`DUAL_ONLY` rotation requires both current keys. Algorithm deprecation never restores classical only authorization.

### **Funding Before Binding and Dust Griefing**

The deterministic `pq_address` path has an unavoidable conflict under a 20 byte address space.
If the balance check were removed, a party that found a truncated key collision could bind and take over a funded unbound EOA. 
If the balance check is retained, any third party can permanently disqualify a published but unbound derived address by transferring one wei to it. 
The intended post-quantum key holder cannot spend the dust or use a classical signature to clear it. There is no recovery path in this SIP.
Atomic initial funding alone prevents accidental pre-funding but does not make a publicly pending deterministic registration safe. 
The pending transaction reveals the public key and address, allowing a proposer or observer with ordering power to place a dust transfer first. 
The address must remain undisclosed until ordering is fixed, or the user must use `CreatePQAccountTx`.

`CreatePQAccountTx` avoids permanent bricking by selecting the first eligible candidate during execution and by funding only after the binding is installed. An adversary can occupy candidates and force a retry, but cannot cause the user to accept a permanently unusable address.

### **Address Truncation**

Post-quantum derived addresses are 20 bytes because this SIP preserves the EVM address space. 
A targeted preimage attack against a 160 bit truncated address has generic cost about `2^160` classically and `2^80` with ideal Grover search. 
Domain separation does not change that bound.
The derivation path therefore requires the target to be empty, and sponsored registration allows the binding to be created before the account is funded. 
Once a binding exists, a colliding public key cannot replace it without satisfying the current authorization and migration nonce rules.

The 80 bit generic quantum bound is one reason this SIP is a transition mechanism rather than the final post-quantum address design.

### **Lost or Compromised Post-Quantum Keys**

This SIP deliberately provides no social, governance, or classical key recovery after an account has entered `PQ_ONLY`. A lost key can permanently lock the account. 
A compromised key can authorize theft and rotation.

Wallets and custodians should test the post-quantum key before registration, maintain protected backups, and rotate on suspected compromise. 
`DUAL` can reduce single key compromise risk before `H_Q`, but it cannot provide post-cutoff recovery because the classical factor is deliberately retired.

### **Algorithm Failure**

Algorithm agility is useful only if users can rotate before a verifier is disabled. 
`G_deprecate` provides a minimum window. Emergency cryptanalytic failure may force governance to choose between continued exposure and account lockup. 
This SIP chooses no automatic weaker fallback. Deprecating the sole live algorithm would lock every binding at the deadline and halt native EOA onboarding. 
The consensus rule requiring a live replacement prevents governance from entering that state through the ordinary deprecation mechanism.

### **Malformed Inputs and Consensus Splits**

Post-quantum decoders have a larger malformed input surface than ECDSA. Clients must use strict canonical parsing and shared negative test vectors.
Library defaults are not a consensus specification.

An implementation must not skip norm checks, normalize a malformed signature into an accepted form, or accept multiple encodings of the same key or signature.

### **Denial of Service**

An attacker can claim a funded sender and attach an invalid post-quantum signature. 
Because the signature is invalid, the claimed account cannot be charged. 
Cheap state and syntax checks, explicit byte limits, peer rate limits, verification caching, and calibrated gas are all required.

The persistent registry is also a state growth surface. Binding creation must be charged for the full public key and metadata, and the chain must monitor registry growth.
Assigned address creation adds a bounded state probing surface. A party can dust predicted candidates to increase probe costs or make one attempt fail. 
Every probe is charged, the loop is bounded by `MaxPQCreateProbes`, and changing `creation_salt` gives a fresh candidate sequence.

### **Registry State Growth**

The persistent registry is a state-growth surface. Binding creation must be charged for the full public key, metadata, and authenticated-state overhead, and the chain must monitor registry growth. A million ML-DSA-44 bindings already imply at least 1.33 GB of raw active-key payload before database amplification.

Bindings cannot be deleted or reduced to key hashes while native validators still need the full key for direct verification. A future authenticated key table with transaction witnesses or proof-batched verification may change that tradeoff, but it requires an explicit state-availability and verification design.

### **Signing Implementations**

ML-DSA signing handles secret dependent values and randomness. 
Wallet and hardware implementations should use reviewed constant time code and the deterministic or hedged signing procedures permitted by FIPS 204. 
Seed backup is acceptable only when key expansion is deterministic and implemented consistently.

### **Throughput and Availability**

A strict block authentication byte limit protects propagation and storage but also caps post-cutoff EOA throughput. 
Governance must not activate `H_Q` under capacity assumptions that were measured only with 65 byte ECDSA signatures.
This SIP does not rely on future lattice signature aggregation. 
The concrete alternatives are a smaller standardized signature profile, proof based batching of verification, or a validity proof authorization layer.
Alternatives that are throughput preserving are an active area of research and will be proposed in a later SIP at an as yet undefined time.

### **Cutoff Governance**
The global cutoff affects every unbound EOA and can render substantial state unable to originate transactions. 
A process recommendation is not sufficient protection, `G_cutoff_notice` is therefore enforced by consensus for the first finite schedule and for every acceleration. 
Reducing the notice parameter is itself delayed under the old value.
The notice window does not guarantee that every holder migrates. 
It prevents ordinary governance from changing the authorization boundary with no protocol enforced migration interval.

### **Classical EIP-7702 Cross-Chain Scope**

The original EIP-7702 classical tuple permits `chain_id = 0`. This broad authorization remains available in modes that accept the classical tuple, including when that tuple is carried inside a post-quantum outer transaction. The post-quantum security of the outer sender does not narrow the classical authority's signed scope. Extended tuples require a nonzero current chain ID.

### **Post-Cutoff Sponsor Dependency**

A user with no existing valid account cannot pay for its first native post-quantum account after `H_Q`. It needs an existing post-quantum payer, a protocol fee grant, or an external sponsor. Sponsor unavailability can therefore become an onboarding availability failure even when the cryptography and registry are working correctly.

### **Other Classical Authorization**

Disabling native EOA ECDSA does not disable classical signa tures inside contracts, bridges, multisignature systems, validator consensus, or application specific authorization. 
Those systems can remain takeover paths even when the originating EOA is post-quantum.

## **References**

- National Institute of Standards and Technology, FIPS 204, *Module Lattice Based Digital Signature Standard*, 2024.
- EIP-2, *Homestead Hard fork Changes*.
- EIP-1559, *Fee Market Change for ETH 1.0 Chain*.
- EIP-2718, *Typed Transaction Envelope*.
- EIP-7702, *Set Code for EOAs*.
- EIP-7932, *Secondary Signature Algorithms*, draft.
- National Institute of Standards and Technology, *FIPS 206 Status Update*, September 2025.
- EIP-191, *Signed Data Standard*.
- EIP-8051, *Precompile for ML-DSA Signature Verification*, draft.
- EIP-8164, *Native Key Delegation for EOAs*, draft.
