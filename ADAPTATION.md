# PLEA Sepolia test run 8.5 adaptation

Run 8.5 rehearses run 8's final code once more with the lighter default build (the suite in `test/full/`, the forks in `test/fork/`). The checkout arrived at `af57f6d` with the five manifest contracts (PLEA, CabalGate, Seasons, PleaDistributor, PleaLaunch), the committed Letterbox and Tribunal, `launch.json` with run 8's constructor words and code hashes, and the fixes of audit rounds 5 to 8 (`audit/FINDINGS.md` R5-* to R8-*). This step audited that code against the import audit's two findings and the launch rules, and **changed no contract, test, script, manifest or build setting**: the deployed bytecode, the Letterbox hash `0xbd93782a…2b4c` and the Tribunal hash `0x48692674…7274` are run 8's. The only files written are this one, `audit/FINDINGS.md` (two rows and a round entry) and `audit/THREAT-MODEL.md` (two wording changes), all explained below. No deployment, signing or broadcast was performed; the earlier adaptations' notes (runs 5 to 8) live in git history and in `audit/FINDINGS.md`.

## The import audit's findings

### 1. Oracle callbacks do not compare `a.requestId` with `intakeId` (Low, id `fa3cd1b8…56ae`): does not reproduce on the live Intake, left alone

**The claim.** `CabalGate.onOracleResult` (`src/CabalGate.sol:530-560`) and `Tribunal.onOracleResult` (`src/Tribunal.sol:308-340`) look the plea or objection up by the callback's `intakeId`, verify the attestation in their own EIP-712 domain, consume `a.requestId` once and apply `a.answer`. Neither compares `a.requestId` with `intakeId`, and `a.questionHash` is never read. The audit's stated precondition is that a third party can complete request B on the live Intake with the attestation issued for request A.

**Reproduced as described, on the mock only.** On the repository's permissionless `MockIntake` (`test/utils/Mocks.sol`, `complete` has no caller check) the delivery order the audit gives settles plea B with plea A's signed score; `test/full/OracleReplay.t.sol::testFuzz_oneSignedRequestCannotSettleTwoPleas` drives exactly that order and passes (one consumption, the second delivery refused `AlreadyConsumed`). So the contract-level observation is correct: the gate and the Tribunal do not bind the signed answer to the request on chain.

**The precondition fails on the live Intake.** The audit could not read the Sepolia Intake v2 at `0xa43e6F75ee006411F79Ac1C84120606C2330DE82`. This step read it: Sourcify holds a full match of its creation and runtime code (match id 55895432, verified 2026-10-10), and its `complete(bytes32 requestId, uint8 status, bytes32 resultHash, string uri, bytes args)` begins with `if (msg.sender != writer) revert NotWriter();`. `writer()` on Sepolia answers `0x3d3C0Df7C1a17d384545e9D7C8228897bD9625CB`, the oracle plane's key. Nobody else can complete a request, so no third party can present attestation A under plea B's id. For a status-0 answer the Intake forwards `args` to the callback as they are (it does not prepend its own request id; it checks the first two words only on a failure), so both `intakeId` and `a.requestId` are the writer's words: the pairing of a signed answer with the request it answers is the writer's, and the writer is the same IMD oracle plane whose signer the threat model already trusts for the answer (`audit/THREAT-MODEL.md` §1). A dishonest writer could at most re-address a signed score; it cannot forge one, and it cannot revive a plea after its 20-minute pending expiry.

**The suggested comparison cannot be added without breaking every live verdict.** The plane's attestation `requestId` is its own request UUID, not the Intake's `keccak256(abi.encode(block.chainid, address(this), ++nonce))`: the live attestation `f7af4af1-b840-4649-9135-283a31158847` kept in `test/full/OracleConformance.t.sol` carries `requestId 0xf7af4af1b840464991352…0000`, and the repository's fixtures model the same split (`attestation(score, keccak256("oracle", id))` against the mock Intake's own request ids). An `a.requestId == intakeId` check would therefore refuse every live answer and kill the Cabal through the 33-hour dead-man. Keying on `a.questionHash` would need the plane's canonical hashing of the question, which is not published; the gate cannot re-hash its question to it.

**Decision.** The finding is recorded as `R85-I-1`, accepted with this reason, in `audit/FINDINGS.md`. Invariant 4 and the Intake's row in `audit/THREAT-MODEL.md` §1 are reworded to say where the binding lives (the writer, not an on-chain check), since the old wording let a reader believe the contract checked it. Round 2's `R2-A1-3` recorded the same observation as an Info and accepted it. No contract changed; a change here would alter run 8's hashes for a check that cannot hold against the live protocol.

### 2. Coverage note (Info, id `5efb5948…8fae5`): not a defect, noted

Recorded as `R85-I-2`. This step reached the live Intake's verified source and its `writer()` (above) and ran the suites listed under Verification. The other live dependencies (PoolManager, Stacker, TestIMD, the oracle fee asset, the seat NFT) are still represented by the repository's mocks and the fork suites, which skip without their RPC variables; no live-fork result is claimed.

## Launch rules checked against the code (nothing to change)

- **Constructors.** PLEA (1 argument), CabalGate (7), Seasons (4), PleaDistributor (2) and PleaLaunch (13) are nonpayable, take only static words, and call no other contract during construction except the earlier contracts of the same transaction (PleaLaunch reads PLEA's owner and registers the system); `test/LaunchManifest.t.sol::test_factoryDeploysExactManifestOnAnEmptySepoliaChain` deploys the manifest on an empty chain. No initializer, proxy, DELEGATECALL, CALLCODE or SELFDESTRUCT (the same test scans the runtime). Argument counts are under 16.
- **Owners.** PLEA and PleaDistributor take `$owner`; PleaLaunch derives its owner from PLEA; Letterbox, Tribunal, CabalGate and Seasons are ownerless; no `msg.sender` or `tx.origin` owner. The PLEA owner's allowlist is add-only and the transfer restriction ends for good at the dead-man, as the threat model records.
- **Hashes and sizes.** `launch.json`'s 12th and 13th PleaLaunch words equal `keccak256` of this build's Letterbox and Tribunal creation code (`test_launchJsonCommitsToThisLetterbox`, `test_launchJsonCommitsToThisTribunal`). Runtime sizes from `out/`: PleaHook 24,402 bytes (174 under EIP-170), CabalGate 24,038, Letterbox 23,255, Tribunal 21,654, Seasons 12,473, PLEA 4,893, PleaDistributor 1,513, PleaLaunch 4,178.
- **Numbers.** Panel 30, quorum 17, score 0-45, need 71 minus the fact score with the run-8 table, letters 20%, plea 1% to 50% or 2.5M, 0.70 IMD minimum, 95% sale floor after the hook's 2% fees, fee cap +10%, seasons 18,000 s, dead-man 118,800 s, START 1792335600, credit gas 1,300,000, objection window 20 minutes, stake 0.35 IMD, margin 2, two objections per plea from different wallets, fees 2.0% split 0.5 / 0.75 / 0.25 / 0.25 / 0.25, the 1.2 seat multiplier, cap decay 15 days, 82/6/12, six 10,000,000 PLEA bonuses, top 25 from 450 to 350 bps: all as they arrived, none touched.
- **Configuration.** `foundry.toml` (the default profile's skip list, `lint_on_build = false`, the read-only `fs_permissions`), `remappings.txt`, `lib/` and `launch.json` are unchanged.

## Verification

Run with the project's own configuration and `FOUNDRY_FUZZ_SEED=0x8f689f5e856863cbf939e08e2376cfb8b2e17a70c2cda8984b3b1c92fefd9ae3`.

| Check | Result |
|---|---|
| `forge build` (default profile) | passed; the three pre-existing warnings (Tribunal `status` parameter names, Seasons unused tuple member, CabalGate `_text` mutability) |
| `forge test` (default profile: Launch, LaunchManifest, ManifestInput) | 21 passed, 0 failed |
| `FOUNDRY_PROFILE=full forge test --match-path 'test/full/{OracleReplay,OracleConformance,TribunalOracleBoundary}.t.sol'` | 17 passed, 0 failed |
| `FOUNDRY_PROFILE=full forge test --match-path 'test/full/{Gate,GateAdversarial,Tribunal,TribunalAdversarial}.t.sol'` | 113 passed, 0 failed |
| Launch gas (from `test_factoryDeploysExactManifestOnAnEmptySepoliaChain`) | under the 16,777,216 cap with the factory's 22% headroom, as the test asserts |

The fork suites were not run (no RPC variables). The rest of the full suite is run 8's, unchanged since its last full pass recorded in `audit/FINDINGS.md`.
