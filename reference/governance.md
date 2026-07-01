# Governance and Security Model

Acta separates cold governance from hot operations. The cold authority controls protocol configuration and the timelock; the hot wallet runs routine market and oracle operations within those limits. The guardian is an emergency canceller/pause key.

## Roles

### Cold authority (`ACTA_ADMIN`)

The cold authority is the hardcoded public key `CLSYf1AL9rXYjGbSjvexMRegriqpLR37kmLkcqvAgBnN`. In production this is intended to be a Squads v4 vault PDA. Acta authorization is a pure signer check: if the transaction signer equals `ACTA_ADMIN`, the cold action is authorized. Squads supplies that signer via its own M-of-N approval flow.

Cold authority operations:
- initialize the global config
- initialize, update, and close premium configs through the timelock
- register makers through the timelock
- create and update oracle sources through the timelock
- raise or keep the global timelock delay immediately
- lower the global timelock delay only through the timelock
- rotate the hot wallet immediately
- set, rotate, or clear the guardian immediately
- toggle pause immediately
- queue and execute pending actions
- cancel pending actions

The cold authority is also the program upgrade authority.

### Hot wallet (`GlobalConfig.hot_authority`)

The hot wallet is the online key used by backend services. It is stored in `GlobalConfig.hot_authority` and can be rotated by the cold authority without a redeploy.

Hot wallet operations:
- create, finalize, and close markets
- create and close concrete oracle accounts from already-approved oracle sources
- publish a post-expiry scalar settlement price to an oracle account (`UpdateOraclePrice` checks the live `hot_authority`; the cold authority is break-glass)
- co-sign `OpenPosition`
- withdraw collected protocol fees

The hot wallet cannot modify cold configuration, rotate itself, bypass maker quote signatures, or bypass taker transaction signatures.

### Guardian (`GlobalConfig.guardian`)

The guardian is an optional emergency key. It defaults to unset; the default public key does not authorize anyone. The cold authority sets, rotates, or clears it with `SetGuardian`.

Guardian operations:
- cancel any queued pending action before execution
- engage the emergency pause (releasing it / unpause is cold-only)

The guardian cannot queue or execute pending actions, rotate itself, rotate the hot wallet, unpause, or perform hot/cold configuration changes. A guardian is a single trusted key: if compromised it can only grief (veto-spam queued actions, re-engage the pause) and the cold authority recovers by rotating it with `SetGuardian` (immediate).

### Permissionless exits

Settlement and liquidation are permissionless. Any signer may settle an expired position or liquidate an ITM unfunded position. Pause does not block exits.

## Authority Matrix

| Action | Authority | Timing |
|--------|-----------|--------|
| Program upgrade | Cold upgrade authority | Outside Acta timelock |
| Initialize global config | Cold | Immediate |
| Rotate hot wallet | Cold | Immediate |
| Set / rotate / clear guardian (`SetGuardian`, op 27) | Cold | Immediate |
| Engage emergency pause (`SetPause`, op 28) | Cold or guardian | Immediate |
| Release emergency pause / unpause (`SetPause`, op 28) | Cold | Immediate |
| Queue pending action (`QueuePendingAction`, op 23) | Cold | Immediate queue |
| Execute pending action (`ExecutePendingAction`, op 25) | Cold | After delay |
| Cancel pending action (`CancelPendingAction`, op 24) | Cold or guardian | Before execute |
| Raise / keep timelock delay (`UpdateActionTimelock`, op 26) | Cold | Immediate |
| Lower timelock delay (`UpdateActionTimelock`, op 26) | Cold | Timelocked by current delay |
| Initialize / update / close premium config | Cold | Timelocked |
| Register maker | Cold | Timelocked |
| Create / update oracle source | Cold | Timelocked |
| Create / finalize / close market | Hot wallet | Immediate |
| Create / close concrete oracle | Hot wallet | Immediate |
| Publish settlement price | Hot wallet (live) or cold | Immediate after expiry |
| Withdraw protocol fees | Hot wallet | Immediate |
| Open position | Hot wallet + taker tx signer + maker Ed25519 quote | Immediate |
| Settle position | Permissionless | After market finalization |
| Liquidate position | Permissionless | ITM + unfunded |

## Timelock

Acta uses one global delay: `GlobalConfig.timelock_secs`. It starts at `0`, should be raised after deployment, and is capped at 7 days. There are no per-kind delay slots and no `GlobalConfig.timelock_secs[0..N]` array.

The timelock uses a generic store-and-replay model. Queue stores a commitment to the exact wrapped instruction; execute replays the same wrapped instruction after `execute_at`.

Timelockable wrapped opcodes:

| Opcode | Instruction |
|--------|-------------|
| 0 | `InitializeConfig` |
| 1 | `UpdateConfig` |
| 2 | `RegisterMaker` |
| 17 | `CreateOracleSource` |
| 18 | `UpdateOracleSource` |
| 20 | `CloseConfig` |
| 26 | `UpdateActionTimelock` |

In production, the first six reject direct calls with `DirectCallDisabled`; they must go through queue -> execute. `UpdateActionTimelock` is special: raising or keeping the delay is immediate; lowering it must be queued and waited out under the current delay.

### Wire Shape

`QueuePendingAction` data is `[23, wrapped_opcode, ...wrapped_args]` and accounts are `[admin, pending_action_pda, system_program, global_config_pda, ...wrapped_accounts]`.

`ExecutePendingAction` data is `[25, wrapped_opcode, ...wrapped_args]` and accounts are `[admin, pending_action_pda, ...wrapped_accounts]`.

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

## Squads Integration

`ACTA_ADMIN` is a Squads **3-of-4** vault PDA, which requires no contract changes. Squads handles member approval and key rotation; Acta checks only the resulting signer. Acta's timelock adds an on-chain, cancellable delay for sensitive actions. The guardian can veto queued actions independently of the proposer.

## Program Upgrade

The program upgrade authority is the same Squads **3-of-4** vault as `ACTA_ADMIN`. A program upgrade can replace any logic — including the timelock and guardian — and is intentionally **not** behind Acta's on-chain timelock or guardian veto. The control on it is the 3-of-4 multisig threshold: an upgrade cannot execute without 3 of the 4 members. There is no separate on-chain upgrade delay.

## Not Implemented

- No per-kind timelock array.
- No `AddPremiumToken` instruction. Supported premium mints are represented by premium config PDAs created through `InitializeConfig`.
- No global-config close instruction. `CloseConfig` closes a premium config.
- No on-chain Squads threshold introspection. The threshold lives in Squads state.
