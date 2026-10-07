# Governance and Security Model

Acta separates cold governance from hot operations. The cold authority controls protocol configuration and the timelock; the hot wallet runs routine market and oracle operations within those limits. The guardian is an emergency canceller/pause key.

## Roles

### Cold authority (`ACTA_ADMIN`)

The cold authority is the hardcoded public key `CLSYf1AL9rXYjGbSjvexMRegriqpLR37kmLkcqvAgBnN`. Squads v4 supplies the vault PDA's signature after its configured approvals. Acta verifies the `ACTA_ADMIN` signature; membership, approval thresholds and Squads execution delays are enforced by Squads.

Cold authority operations:
- initialize the global config
- initialize, update, and close premium configs through the timelock
- register makers through the timelock
- create and update oracle sources through the timelock
- raise or keep the global timelock delay immediately
- lower the global timelock delay only through the timelock
- rotate the hot wallet immediately
- set, rotate, or clear the guardian immediately
- set, rotate, or revoke the settlement attestor through the timelock
- toggle pause immediately
- queue pending actions
- cancel pending actions

The cold authority is separate from the program upgrade authority.

### Hot wallet (`GlobalConfig.hot_authority`)

The hot wallet is the online key used by backend services. It is stored in `GlobalConfig.hot_authority` and can be rotated by the cold authority without a redeploy.

Hot wallet operations:
- create, finalize, and close markets
- create and close concrete oracle accounts from already-approved oracle sources
- publish a post-expiry scalar settlement price to an oracle account. `UpdateOraclePrice` checks the live `hot_authority`, and works for the hot wallet only while no settlement attestor is configured: once `GlobalConfig.settlement_attestor` is set, direct hot publication is rejected on-chain (`SettlementAttestationRequired`, 1090) and settlement requires `UpdateOraclePriceAttested` with an Ed25519 attestation from that key — a 2-of-2 of hot key and attestor key. The cold authority retains a break-glass direct path
- co-sign `OpenPosition`
- withdraw collected protocol fees

The hot wallet cannot modify cold configuration, rotate itself, bypass maker quote signatures, or bypass taker transaction signatures.

### Guardian (`GlobalConfig.guardian`)

The guardian is an optional emergency key. It defaults to unset; the default public key does not authorize anyone. The cold authority sets, rotates, or clears it with `SetGuardian`.

Guardian operations:
- cancel any queued pending action before execution
- engage the emergency pause (releasing it / unpause is cold-only)

The guardian cannot queue pending actions, rotate itself, rotate the hot wallet, unpause, or authorize hot/cold configuration changes. A guardian is a single trusted key: if compromised it can only grief (veto-spam queued actions, re-engage the pause) and the cold authority recovers by rotating it with `SetGuardian` (immediate).

### Permissionless exits

Settlement and liquidation are permissionless. Any signer may settle an expired position or liquidate an ITM unfunded position. Pause does not block exits.

## Authority Matrix

| Action | Authority | Timing |
|--------|-----------|--------|
| Program upgrade | Separate program-upgrade authority | Its configured Squads approvals and delay |
| Initialize global config | Cold | Immediate |
| Rotate hot wallet | Cold | Immediate |
| Set / rotate / clear guardian (`SetGuardian`, op 27) | Cold | Immediate |
| Engage emergency pause (`SetPause`, op 28) | Cold or guardian | Immediate |
| Release emergency pause / unpause (`SetPause`, op 28) | Cold | Immediate |
| Queue pending action (`QueuePendingAction`, op 23) | Cold | Immediate queue |
| Execute pending action (`ExecutePendingAction`, op 25) | Any signer | After the cold-approved action's delay |
| Cancel pending action (`CancelPendingAction`, op 24) | Cold or guardian | Before execute |
| Raise / keep timelock delay (`UpdateActionTimelock`, op 26) | Cold | Immediate |
| Lower timelock delay (`UpdateActionTimelock`, op 26) | Cold | Timelocked by current delay |
| Initialize / update / close premium config | Cold | Timelocked |
| Register maker | Cold | Timelocked |
| Create / update oracle source | Cold | Timelocked |
| Create / finalize / close market | Hot wallet | Immediate |
| Create / close concrete oracle | Hot wallet | Immediate |
| Publish settlement price (`UpdateOraclePrice`, op 16) | Hot wallet (only while no attestor is set) or cold | Immediate after expiry |
| Publish attested settlement price (`UpdateOraclePriceAttested`, op 31) | Hot wallet + attestor Ed25519 proof (2-of-2) | After expiry, attestor set |
| Set / rotate settlement attestor (`SetSettlementAttestor`, op 29) | Cold | Timelocked |
| Revoke settlement attestor (`RevokeSettlementAttestor`, op 30) | Cold | Timelocked |
| Withdraw protocol fees | Hot wallet | Immediate |
| Open position | Hot wallet + taker tx signer + maker Ed25519 quote | Immediate |
| Settle position | Permissionless | After market finalization |
| Liquidate position | Permissionless | ITM + unfunded |

## Timelock

Acta uses one global delay for timelocked actions: `GlobalConfig.timelock_secs`, from `0` to 604,800 seconds (7 days). Its initial value is `0`. Zero permits immediate execution of a queued action; cold authorization and the instruction commitment still apply.

The timelock uses a generic store-and-replay model. Queue stores a commitment to the exact wrapped instruction; execute replays the same wrapped instruction after `execute_at`.

Cold authorizes the action when queueing it. Once its delay has elapsed, any signer can execute that exact action. The executor pays transaction costs and funds accounts created during execution; closing the pending-action account returns its rent to the original proposer.

Supported premium mints are configured through premium config PDAs. `InitializeConfig` creates a premium config, `UpdateConfig` updates it, and `CloseConfig` closes that premium config.

Timelockable wrapped opcodes:

| Opcode | Instruction |
|--------|-------------|
| 0 | `InitializeConfig` |
| 1 | `UpdateConfig` |
| 2 | `RegisterMaker` |
| 17 | `CreateOracleSource` |
| 18 | `UpdateOracleSource` |
| 20 | `CloseConfig` |
| 29 | `SetSettlementAttestor` |
| 30 | `RevokeSettlementAttestor` |
| 26 | `UpdateActionTimelock` |

In production, all listed operations except the immediate raise/keep form of `UpdateActionTimelock` reject direct calls with `DirectCallDisabled`; they must go through queue -> execute. `UpdateActionTimelock` is special: raising or keeping the delay is immediate; lowering it must be queued and waited out under the current delay.

### Wire Shape

`QueuePendingAction` data is `[23, wrapped_opcode, ...wrapped_args]` and accounts are `[admin, pending_action_pda, system_program, global_config_pda, ...wrapped_accounts]`.

`ExecutePendingAction` data is `[25, wrapped_opcode, ...wrapped_args]` and accounts are `[executor, pending_action_pda, proposed_by, ...wrapped_accounts]`.

The commitment binds:

```
sha256(
  "acta:pend.v2"
  || wrapped_opcode
  || ordered wrapped account keys
  || full wrapped instruction data
)
```

Client note: generated Codama builders for `QueuePendingAction` and `ExecutePendingAction` expose only the wrapped `opcode`, so they are not sufficient for real generic timelock payloads. Use the SDK flow helpers or manually build the full wrapped wire layout above.

## Emergency Pause

`SetPause` sets the `PAUSED` bit in `GlobalConfig.flags`. **Engaging** the pause is immediate for cold or guardian; **releasing** it (unpause) is **cold-only**, so a compromised guardian cannot unpause to resume an attack.

While paused, `OpenPosition` fails with `ProtocolPaused`. Settlement, liquidation, and other exit paths remain available.

## Settlement Attestation

`GlobalConfig.settlement_attestor` optionally names an independent Ed25519 second signer for hot settlement publication. While it is unset (all-zero), `UpdateOraclePrice` behaves as described above. Once set (`SetSettlementAttestor`, cold-only), the hot wallet's direct call fails with `SettlementAttestationRequired` (1090); settlement then requires `UpdateOraclePriceAttested`, where an Ed25519 verify instruction signed by the attestor over a domain-separated message must immediately precede the update in the same transaction. The cold authority keeps the direct path as break-glass (Ed25519 precompile instructions cannot be produced through Squads CPI). `RevokeSettlementAttestor` (cold-only) clears the key and deliberately returns hot settlement to direct mode; the 32-byte replay domain is retained so signatures can never be replayed across deployments.

## Squads Integration

Squads handles approvals and member rotation; Acta checks the cold vault signature. The approval threshold and Squads delay are stored in the multisig account. The Acta timelock separately controls when queued protocol actions become executable, and the guardian can cancel them before execution.

## Program Upgrade

Program upgrades use a separate Squads vault and its configured approvals and execution delay. The program's upgrade authority authorizes deployment of the program buffer. `GlobalConfig.timelock_secs` applies to Acta's queued instructions, not to program upgrades.
