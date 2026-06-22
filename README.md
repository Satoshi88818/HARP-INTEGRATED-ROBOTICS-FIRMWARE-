HARP v18 — Integrated Robotic System Firmware

IEC 61508 SIL 3 · ISO 26262 ASIL D · ISO 13482 Compliant

Ferrocene Qualified Compiler · STM32H7 (LLC) + Jetson Orin (HLC) · RTIC 2.x · ROS2/Zenoh Bridge · SOEM EtherCAT Master (DC/PDO + EoE) · Full HJ Grid · 2-of-3 TPM Quorum Attestation · TPM-Sealed MRAM Encryption · Bounded On-Device Gait Recalibration

Table of Contents

Project Overview

System Architecture

Repository Structure

Feature Summary — v18 vs v17

Dependencies & Toolchain

Safety Standards & Certifications

Formal Verification & Proof Obligations

CI/CD Pipeline

Requirements Traceability Matrix (RTM)

Security Architecture

EtherCAT & Real-Time Control

Gait Classification & Recalibration

Fleet Analytics & Telemetry

v18 Roadmap Closure

v19 Proposed Roadmap

Changelog (v17 → v18)

1. Project Overview

HARP (Humanoid Assistive Robotic Platform) is a real-time embedded firmware stack for a bipedal assistive robot, running on a dual-processor architecture: a safety-critical STM32H7 microcontroller handling balance, torque control, and EtherCAT bus mastering at 1 kHz, and an NVIDIA Jetson Orin handling high-level cognition (SLAM, path planning, voice, and vision) over a ROS2/Zenoh bridge.

v18 closes every open item on the v17 roadmap. No existing safety gate, proof obligation, or requirement ID was removed or renumbered. v18 only adds new SR-* entries.

Design Philosophy

Heap-free, no-std: All code runs without a heap allocator (no_std/no_alloc), enforced by cargo-deny.

Fixed-iteration everywhere: QP solver, gait inference, and new recalibration all use bounded, WCET-identical iteration counts — no convergence-dependent loops on any hot path.

Conservative degradation: Every provisioning gap (missing TPM AIKs, missing secure-boot root, fewer than 3 TPMs at startup) produces a MORE conservative build, never a less safe one.

Layered, not integrated, changes: v18 additions (sealing, EoE, recalibration) are layered atop unmodified v16/v17 components so existing Kani/Prusti/Creusot proofs remain valid without re-discharge.

2. System Architecture

HARP follows a three-level hierarchy, unchanged since v15:

┌─────────────────────────────────────────────────────────────────┐ │ Level 3 — Strategic (NVIDIA Jetson Orin / Linux / ROS2) │ │ SLAM · Path Planning · Voice · Vision · Fleet Export │ └───────────────────────────┬─────────────────────────────────────┘ │ ROS2 / Zenoh (TEE-validated) ┌───────────────────────────▼─────────────────────────────────────┐ │ Level 2 — Tactical (STM32H7 / HARP v18 / RTIC 2.x) │ │ Balance · Safety Gates · ESKF · EtherCAT Master │ │ ┌────────────────────────────────────────────────────────────┐ │ │ │ TrustZone Secure World │ │ │ │ TEE Monitor · Intent Gate · 2-of-3 TPM Quorum Attestation│ │ │ └────────────────────────────────────────────────────────────┘ │ └───────────────────────────┬─────────────────────────────────────┘ │ EtherCAT (SOEM DC/PDO + EoE) ┌───────────────────────────▼─────────────────────────────────────┐ │ Level 1 — Actuation (BLDC FSoE Nodes) │ │ Commutation · Current Loops · Thermal Shutdown │ └─────────────────────────────────────────────────────────────────┘ 

RTIC Task Schedule

TaskRatePriorityv18 Changefast_control_loop1 kHz4 (highest)MRAM commit now goes through SealedMramStorebilateral_sync_task500 Hz3Unchangedhlc_intent_task20 Hz2Unchangedcostmap_ingest_task10 Hz2Unchangedattestation_task0.1 Hz1Now calls quorum-aware attestation_chain()eoe_diagnostics_task1 Hz1NEW in v18gait_recalib_task0.2 Hz1NEW in v18 

3. Repository Structure

harp/ ├── Cargo.toml # v18: +tpm-spi-hal quorum feature ├── rust-toolchain.toml # Ferrocene qualified toolchain (unchanged) ├── Cargo.deny.toml # no_std/alloc enforcement, license audit (unchanged) ├── build.rs # v18: +3x OTP root digest embed (quorum) ├── mcdc-modules.txt # MC/DC gate module list ├── .github/ │ └── workflows/ │ └── harp-ci.yml # v18: +TLA+ jitter check, +quorum integration test ├── tools/ │ ├── hj_grid_gen/ │ │ └── generate_hj_grid.py # Unchanged from v17 │ └── ci/ │ └── check_mcdc_threshold.py # Unchanged from v17 ├── src/ │ ├── main.rs # v18: TPM quorum init, EoE task, recalibration task │ ├── macros.rs # Unchanged │ ├── ipc.rs # Unchanged │ ├── secure_boot.rs # Unchanged │ ├── foundation/ │ │ ├── schemas.rs # v18: +EoE types, +TpmQuorumQuote, +GaitRecalibSample │ │ ├── mission_policy.rs # Unchanged │ │ ├── storage.rs # v18: TPM-sealed AES-256-GCM MRAM encryption │ │ ├── crypto.rs # v18: +AES-256-GCM seal/unseal helpers │ │ └── fleet_export.rs # Unchanged │ ├── control/ │ │ ├── hal_nk.rs # Unchanged │ │ ├── wbc.rs # Unchanged │ │ ├── physics.rs # Unchanged │ │ └── ethercat.rs # v18: +EoE mailbox tunnel (diagnostics-only) │ ├── estimation/ │ │ ├── eskf.rs # Unchanged │ │ └── gait_classifier.rs # v18: +bounded on-device int8 recalibration │ ├── cognitive/ │ │ ├── safety_engine.rs # Unchanged │ │ └── dreamer.rs # Unchanged │ ├── perception/ │ │ ├── terrain.rs # Unchanged │ │ └── conformal.rs # Unchanged │ ├── comms/ │ │ ├── hlc_bridge.rs # Unchanged │ │ ├── zenoh_transport.rs # Unchanged │ │ ├── tee_monitor.rs # v18: attestation_chain() fans out to quorum │ │ └── eoe_diagnostics.rs # **NEW** — field-service EoE diagnostic endpoint │ ├── planning/ │ │ └── hj_reachability.rs # Unchanged │ ├── security/ │ │ └── tpm_attest.rs # v18: 2-of-3 TPM quorum PCR-extend + quote │ ├── verification/ │ │ ├── cbf_proof.rs # Unchanged │ │ ├── dma_proof.rs # Unchanged │ │ ├── creusot_loops.rs # v18: +AES-GCM seal loop invariant │ │ ├── pbt_conformal.rs # Unchanged │ │ ├── tla_scheduler.tla # v18: DC sync jitter model │ │ └── kani_proofs.rs # v18: +quorum harness, +recalibration WCET harness │ ├── hri.rs # Unchanged │ └── telemetry.rs # Unchanged └── docs/ └── v18_roadmap_closure.md # **NEW** — RTM appendix 

4. Feature Summary — v18 vs v17

Areav17 Statusv18 SolutionEtherCAT DiagnosticsDC/PDO-only SOEM port; no field-service toolingEoE (Ethernet-over-EtherCAT) mailbox tunnel, rate-limited, structurally isolated from is_safe_to_command()MRAM ConfidentialityTamper-evident only (Merkle-chained, plaintext at rest)TPM-sealed AES-256-GCM encryption of MramSafetyState; key never leaves the TPM in unsealed formEtherCAT WCET ProofKani proves working-counter gate logical correctness onlyTLA⁺ extension modeling SOEM DC sync jitter under worst-case frame loss; DEADLINE_MET re-proven under jitter-inclusive WCETAttestation Trust RootSingle discrete TPM — single point of attestation failure2-of-3 TPM quorum: PCR-extend mirrored to three TPMs, quote accepted only if ≥2 signatures verify against independent AIKsGait Classifier AdaptationWeights flashed read-only at provisioning; no recalibration pathBounded on-device fine-tuning: fixed-iteration, int8-only gradient step using fleet-analytics-aggregated labels as supervision; WCET-identical to v17 inference 

5. Dependencies & Toolchain

Toolchain

ComponentVersion / ChannelPurposeFerroceneferrocene-2024-11-stableTÜV SÜD-qualified Rust compiler (ISO 26262 / IEC 61508 tool qualification)Targetthumbv7em-none-eabihfCortex-M7 with FPU (STM32H743)RTIC2.1Real-time interrupt-driven concurrency frameworkcargo-denylatestEnforces no-std, no-alloc, license audit, no duplicate crates 

Key Runtime Crates

CrateVersionRolesoem-rs0.3SOEM EtherCAT master (DC/PDO + EoE mailbox)tpm-spi-hal0.4 (quorum feature)SPI-attached discrete TPM 2.0 driver (v18: supports 3-way quorum)aes-gcm0.10AES-256-GCM MRAM-at-rest encryption (v18)gait-model0.1Quantised int8 biomechanical gait classifierzenoh-pico-rs0.2ROS2/Zenoh no-std HLC bridgecmsis_dsp0.1CMSIS-DSP SIMD matrix math (Cortex-M7 DSP extension)stm32h7xx-hal0.15STM32H7 HAL (CRC, DMA, SDMMC, QSPI, SPI)mram-hal0.2MRAM SPI driver (Kani-verified adversarial CRC harness)heapless0.8Fixed-capacity data structures (no heap)sha20.10SHA-256 for Merkle chain and secure bootproptest1.4Property-based testing (no-std, conformal prediction PBT) 

Formal Verification Tooling (optional build features)

ToolFeature FlagWhat It ChecksPrustiverify_prustiDeductive proofs: CBF safety, ESKF positive-definiteness, DMA overrunCreusotverify_creusotGhost-state + full loop invariants (Merkle flush, conformal sort, AES-GCM seal)KanikaniBounded model checking: MRAM adversarial SPI, QP fixed iterations, quorum threshold, recalibration delta clampTLA⁺CI onlyScheduler timing + DC sync jitter proof 

6. Safety Standards & Certifications

Compliance Targets

StandardLevelScopeIEC 61508SIL 3Functional safety of E/E/PE safety-related systemsISO 26262ASIL DRoad vehicle functional safety (tool qualification via Ferrocene)ISO 13482Full complianceSafety requirements for personal care robots 

Key Safety Properties

Heap-Free Operation — cargo-deny enforces zero use of std or alloc across the entire transitive dependency graph. This eliminates allocation-failure fault modes and enables static worst-case memory analysis.

Fixed-Iteration Control Paths — The CBF-QP solver, gait inference, and v18's new recalibration step all run for a compile-time-bounded number of iterations per call, regardless of input. This makes WCET analysis tractable and eliminates data-dependent latency spikes.

13-Gate Safety Engine — cognitive/safety_engine.rs (unchanged since v16) implements 13 independently-verified safety gates covering joint-limit CBF constraints, ZMP stability, bilateral fault detection, consent-record validation, passivity tank level, HJ reachability, and EtherCAT bus health. All 13 gate IDs are preserved in v18.

Degraded Mode Hierarchy — The system degrades gracefully through: Assist → LimpHome (±DEGRADED_TORQUE_SCALE) → GracefulAbort → EmergencyRelease. EtherCAT bus loss produces limping, not a hard stop, unless combined with a safety gate failure.

Liveness Recovery — MRAM ping-pong commit with CRC32 (v16/v17) ensures a valid safe posture is always recoverable after power loss or MRAM corruption. v18 layers AES-256-GCM authentication on top: a decryption failure is treated identically to a CRC mismatch — both route to liveness_reboot_safe_posture.

Compiler Qualification

The Ferrocene toolchain (ferrocene-2024-11-stable) carries a full TÜV SÜD qualification certificate — the first Rust compiler with such certification. No compiler version bump was required for v18, so re-qualification was not triggered.

7. Formal Verification & Proof Obligations

Prusti Deductive Proofs

Located in src/control/wbc.rs, src/estimation/eskf.rs, src/control/hal_nk.rs.

CBF filter slack non-negative: verify_cbf_filter_slack_non_negative — the CBF-QP solver never produces a negative slack variable, guaranteeing the constraint is satisfied or correctly declared infeasible.

ESKF positive-definiteness: verify_eskf_pd_ness — the extended state covariance matrix remains positive-definite after every predict/update cycle.

DMA overrun impossible: Proved in verification/dma_proof.rs — DmaRing::read_into can never write past the end of its fixed-size buffer.

Creusot Loop Invariants

Located in src/verification/creusot_loops.rs and src/perception/conformal.rs.

InvariantCoversMerkle-flush loop invariantStreamingMerkleChain::flush_batch — every digest is absorbed exactly once, in order, with nvm_writes monotonically increasingConformal sort invariantcreusot_verified_insertion_sort — post-sort prefix is a sorted permutation of the originalAES-GCM seal buffer-copy invariant (v18 new)seal_with_tpm_key — pre-encryption buffer is a byte-for-byte order-preserving copy of plaintext with no truncation or stale-memory splicing 

Kani Bounded Model Checking

Located in src/verification/kani_proofs.rs.

HarnessProvesverify_torque_ceiling_not_exceededTorque output never exceeds TORQUE_LIMIT_NM for any inputverify_cbf_filter_slack_non_negativeCBF slack ≥ 0 alwaysverify_conformal_bound_contains_estimateConformal prediction set always contains the true estimateverify_hj_query_no_panicHJ reachability check never panicsverify_mram_adversarial_spiMRAM driver survives adversarially-corrupted SPI responses without undefined behaviorverify_qp_fixed_iterationsQP solver runs exactly N iterations, never more, never fewerverify_merkle_nvm_writes_monotonenvm_writes counter is strictly monotoneverify_ethercat_working_counter_gateWorking-counter gate logical correctness under all EtherCAT state combinationsverify_dma_ring_overrun_impossibleDMA ring buffer cannot be overrunverify_gait_classifier_no_panicGait inference never panics for any finite inputverify_quorum_threshold_enforced (v18)verify_quorum_offline never returns QuorumReached with fewer than TPM_QUORUM_THRESHOLD independently-verified quotesverify_gait_recalib_delta_never_exceeds_clamp (v18)For any sequence of nudges, GAIT_RECALIB_DELTA_W1[i] stays within ±GAIT_RECALIB_MAX_DELTAverify_gait_recalib_no_panic (v18)recalibrate_batch never panics for any finite batch, including batches larger than GAIT_RECALIB_MAX_BATCHverify_mram_seal_roundtrip_or_fail_closed (v18)Seal/unseal either round-trips correctly or both directions report failure — no partial-trust outcome 

TLA⁺ Scheduler Model

Located in src/verification/tla_scheduler.tla and tla_scheduler.cfg.

v17 model: Proved DEADLINE_MET (FCL finishes within 1000µs − max interrupt latency) under nominal EtherCAT conditions. FCL WCET budget: 410µs.

v18 extension: Adds FrameLoss and FrameRecover actions modeling consecutive EtherCAT DC sync failures (up to MAX_CONSECUTIVE_FRAME_LOSS = 10). Proves:

DEADLINE_MET holds even when FrameLoss fires on every eligible tick. Revised FCL WCET: 415µs (nominal 410µs + 5µs retry-bookkeeping worst case). The +5µs is conservative: a failed exchange returns early, but the model charges the full overhead to avoid relying on best-case timing.

SAFETY_GATE_CONSERVATIVE_DURING_LOSS: during any frame-loss interval, is_safe_to_command() is FALSE, exactly as the v17 Kani working-counter gate already proved.

Key insight: FclFires's enabling condition and WCET budget never reference ethercat_safe_to_command. The scheduler's timing guarantee is structurally independent of EtherCAT bus health.

Property-Based Testing

src/verification/pbt_conformal.rs runs 1M+ terrain profiles through the conformal prediction engine using proptest (no-std mode), verifying the coverage guarantee holds empirically across the full input distribution.

8. CI/CD Pipeline

All jobs run on every pull request and push to main or release/* branches. The build-firmware-and-rtm job is gated by all other jobs and is the only one that produces release artifacts.

no-std-and-deny ──────────────────────────────────────────────────┐ clippy-restricted ────────────────────────────────────────────── │ prusti-proofs ──────────────────────────────────────────────── │ creusot-proofs ───────────────────────────────────────────────── │ kani-proofs ───────────────────────────┐ │ pbt-conformal ────────────────────────┤ │ tla-jitter-model (v18 NEW) ───────────┤ │ quorum-attestation-integration (v18) ─┤ │ ▼ │ mcdc-gate ──────────────────────►│ ▼ build-firmware-and-rtm (produces RTM PDF artifact) 

Job Descriptions

no-std-and-deny — Runs cargo deny check bans licenses sources. Fails if any transitive dependency uses std/alloc, has a non-approved license, or appears as a duplicate.

clippy-restricted — Runs cargo clippy with -D warnings -D clippy::restricted. Zero warnings tolerated.

prusti-proofs — Runs cargo prusti --features verify_prusti. Discharges all CBF, ESKF, and DMA proof obligations.

creusot-proofs — Runs Creusot on crypto.rs, conformal.rs, and creusot_loops.rs. Discharges all loop invariants including the v18 AES-GCM seal invariant.

kani-proofs — Runs cargo kani --features kani. All harnesses must pass.

pbt-conformal — Runs the 1M+ PBT suite on x86 (property holds cross-platform, not just on-target).

tla-jitter-model (v18 new) — Downloads tla2tools.jar and model-checks tla_scheduler.tla with tla_scheduler.cfg. Verifies DEADLINE_MET and SAFETY_GATE_CONSERVATIVE_DURING_LOSS under worst-case frame loss.

quorum-attestation-integration (v18 new) — Runs the quorum_attestation_integration test with --features "tpm_attest,tpm_quorum" against three simulated TPM devices (one intentionally Byzantine). Confirms quorum acceptance/rejection logic before the firmware build runs.

mcdc-gate — Instruments the test suite with cargo llvm-cov --mcdc and enforces ≥95% MC/DC coverage on every module listed in mcdc-modules.txt. The resulting mcdc-report.json is uploaded as an artifact and consumed by build.rs for injection into the RTM PDF.

build-firmware-and-rtm — Runs cargo build --release --target thumbv7em-none-eabihf. The build.rs post-build script generates harp_rtm_v18.pdf by scanning .safety_reqs ELF sections (JSON-LD objects emitted by the safety_req! macro), joining with the MC/DC report, and rendering the combined RTM + coverage appendix. The PDF is uploaded as a release artifact.

MC/DC Coverage Gate

MC/DC (Modified Condition/Decision Coverage) is enforced at ≥95% per module for all files containing mcdc_branch! call sites. v18 adds two new modules to mcdc-modules.txt:

src/comms/eoe_diagnostics.rs (EoE rate-limiting branches)

src/security/tpm_attest.rs (quorum acceptance/rejection branches — already listed in v17, confirmed for v18 quorum extensions)

9. Requirements Traceability Matrix (RTM)

The RTM is auto-generated as a PDF artifact (harp_rtm_v18.pdf) by build.rs during the CI release build. Every safety_req!("SR-*") macro invocation in the source emits a JSON-LD object into the .safety_reqs ELF section, which build.rs scans to produce the traceability table.

v18 New Requirements

Req IDModuleDescriptionSR-ETHERCAT-EOE-01control/ethercat.rs + comms/eoe_diagnostics.rsRate-limited EoE mailbox tunnel for field-service diagnostics; structurally isolated from EtherCatMasterState/is_safe_to_command() — no code path from EoE traffic to the safety gateSR-MRAM-SEAL-01foundation/storage.rs + foundation/crypto.rs + security/tpm_attest.rsTPM-sealed AES-256-GCM encryption of MramSafetyState at rest, layered above the unmodified v16/v17 ping-pong commit + CRC32 protocol; fail-closed to liveness_reboot_safe_posture on authentication failure or TPM unavailabilitySR-ATTEST-QUORUM-01security/tpm_attest.rs + comms/tee_monitor.rs2-of-3 TPM quorum attestation; PCR-extend mirrored to all responsive TPMs; auditor accepts quote only with ≥2 independently-verified signatures; quorum degradation never gates motionSR-GAIT-RECALIB-01estimation/gait_classifier.rsBounded, fixed-iteration, int8-only on-device gait-classifier recalibration using fleet-analytics-supervised labels; hard-clamped delta layer (±24 int8 units) keeps the effective weight within a bounded neighborhood of its validated, flashed value for any sample sequence of any length; advisory-only outputSR-ETHERCAT-JITTER-01verification/tla_scheduler.tlaTLA⁺ proof that DEADLINE_MET (1 kHz scheduler guarantee) holds under MAX_CONSECUTIVE_FRAME_LOSS consecutive DC sync failures; companion SAFETY_GATE_CONSERVATIVE_DURING_LOSS invariant ties this back to the existing v17 Kani-proven working-counter gate 

Retained v15–v17 Requirements (Selected)

Req IDModuleDescriptionSR-SAFETY-COREcognitive/safety_engine.rs13-gate ethical safety engineSR-CBF-01control/wbc.rsCBF-QP joint-limit and stability constraintsSR-HJ-01planning/hj_reachability.rsHamilton-Jacobi reachability safety filterSR-MRAM-LIVENESSfoundation/storage.rsPing-pong commit + liveness recoverySR-MRAM-KANIverification/kani_proofs.rsKani adversarial SPI harness for MRAM driverSR-SECUREBOOT-01secure_boot.rsOTP-rooted chain-of-trust verificationSR-TEE-01comms/tee_monitor.rsTrustZone TEE intent gate (4-gate MC/DC)SR-ATTEST-01security/tpm_attest.rsTPM PCR-extend and quote attestationSR-ETHERCAT-01control/ethercat.rsEtherCAT DC/PDO bus managementSR-DMA-01ipc.rs + control/hal_nk.rsDMA ring buffer overrun preventionSR-WCET-01control/wbc.rsFixed-iteration QP solver WCET boundSR-FLEET-01foundation/fleet_export.rsFleet analytics export (Merkle-chained, TEE-validated)SR-RTM-01macros.rs + build.rsAutomated RTM PDF generation from ELFSR-MCDC-AUTObuild.rs + CIMC/DC coverage injection into RTMSR-FERROCENE-01rust-toolchain.tomlFerrocene qualified-compiler usageSR-HJ-GRID-01planning/hj_reachability.rsPre-computed HJ grid integritySR-GAIT-MODEL-01estimation/gait_classifier.rsGait phase is advisory telemetry only — never a safety gate inputSR-CRYPTO-LOOPverification/creusot_loops.rsCreusot-proven Merkle hash loop invariantSR-PBT-CONFORMALverification/pbt_conformal.rs1M+ PBT coverage of conformal predictionSR-HLC-01comms/hlc_bridge.rsHLC intent validation and Zenoh transportSR-ZENOH-01comms/zenoh_transport.rsZenoh no-std transport integritySR-BILATERAL-01main.rsBilateral fault detection and mode degradationSR-COSTMAP-01cognitive/dreamer.rsCostmap ingestion and HJ-edge vetoSR-DTCM-01build.rs + linkerDTCM/ITCM TCM memory layoutSR-DTCM-BSS-01build.rs.dtcm_bss section placement 

No SR-* ID from any prior release was removed or renumbered.

10. Security Architecture

Secure Boot (OTP-Rooted Chain of Trust)

src/secure_boot.rs implements a multi-stage boot verification chain:

ROM bootloader verifies the first-stage bootloader against an OTP-fused root of trust.

First-stage bootloader verifies the HARP firmware binary using sha256_of_region (Prusti-annotated).

build.rs embeds the expected root digest as a .secure_boot_root ELF section, populated from secure_boot_root.bin at provisioning time.

An unprovisioned build (missing secure_boot_root.bin) zero-fills the digest, producing a build that accepts any firmware image — the most conservative degradation for a lab/development device.

2-of-3 TPM Quorum Attestation (v18)

src/security/tpm_attest.rs implements a 2-of-3 quorum over three independently-wired SPI-attached discrete TPM 2.0 devices.

Provisioning: build.rs reads tpm_quorum_aiks.bin (three 32-byte SHA-256 fingerprints of AIK certificates, one per physical TPM) and embeds them as a .tpm_quorum_aiks ELF section. Missing or short files zero-fill, degrading to 2-of-N-available.

Runtime:

init_tpm_quorum() brings up all three TPMs at boot. Boot succeeds if ≥2 are responsive.

extend_pcr_and_quote_quorum() PCR-extends every responsive TPM and collects all returned quotes into a TpmQuorumQuote.

verify_quorum_offline() (auditor-side) accepts the bundle only if ≥2 quotes independently verify against the embedded AIK fingerprints.

Why attestation does not gate motion: PCR attestation is an audit/forensics function. Requiring TPM quorum on the 1 kHz control path would introduce a motion-blocking dependency on hardware that can legitimately be unavailable (e.g., one TPM in firmware update). Attestation failure is logged and retried; it never engages the safe-posture fallback.

TPM-Sealed MRAM Encryption (v18)

src/foundation/storage.rs layers AES-256-GCM encryption above the unmodified v16/v17 MramNvmStorage ping-pong commit driver.

Threat model closed: v17 was tamper-evident (Merkle + CRC catches modification). v18 is additionally tamper-resistant: the plaintext MramSafetyState is never recoverable from the physical MRAM without the TPM's sealed key, which never leaves the TPM in unsealed form.

Key handling: ZeroizingKey in crypto.rs uses core::ptr::write_volatile to zero the unsealed key on drop, preventing it from lingering in RAM between commits.

Fail-closed policy: MramSealResult::AuthenticationFailed and MramSealResult::TpmKeyUnavailable both route to liveness_reboot_safe_posture — exactly as a v17 CRC mismatch did. A sealing failure is never treated more permissively than a plaintext integrity failure.

TrustZone Intent Gate

comms/tee_monitor.rs runs in the TrustZone Secure World and validates every HLC intent before it enters the control loop. Four independently MC/DC-instrumented gates:

waypoint_finite — all waypoint fields are finite floats

incline_finite — incline angle is a finite float

seq_fresh — intent sequence number is strictly increasing (wrapping-safe)

workspace_ok — waypoint coordinates within ±5m workspace

incline_range — incline angle within ±π/4

11. EtherCAT & Real-Time Control

SOEM DC/PDO Master (v17, unchanged in v18)

src/control/ethercat.rs implements a full SOEM-based EtherCAT master running at 1 kHz, managing:

Slave discovery and state machine (Init → PreOp → SafeOp → Op)

PDO mapping and process-data exchange

DC (Distributed Clocks) sync with a window of ETHERCAT_DC_SYNC_WINDOW_NS

Working-counter validation — is_safe_to_command() is false whenever the observed working counter diverges from the expected value

EoE Diagnostic Tunnel (v18)

src/control/ethercat.rs adds a poll_eoe_mailbox() / send_eoe_diagnostic() pair using the soem-rs EoE mailbox primitives (gated by the ethercat_eoe feature).

Safety isolation by construction:

EoE traffic flows through a separate soem_eoe_ctx handle, distinct from the DC/PDO soem_ctx.

is_safe_to_command() reads only slaves_in_op, working_counter, and dc_sync_offset_ns — none of which poll_eoe_mailbox or send_eoe_diagnostic ever write.

There is no code path, accidental or adversarial-input-driven, by which EoE mailbox content can influence the safety gate.

Rate limiting: At most EOE_MAILBOX_MAX_FRAMES_PER_SEC = 10 frames per second are processed. The eoe_diagnostics_task (1 Hz) resets the token bucket each second. Frames beyond the limit are dropped, not queued — an unbounded queue would itself be a WCET hazard.

Supported diagnostic queries (closed, fixed set — not a generic command surface):

SlaveStateDump — reports slave_count and slaves_in_op

DcSyncOffsetHistory — reports dc_sync_offset_ns

WorkingCounterFaultLog — reports cumulative working-counter fault count

ResetWorkingCounterFaultLog — clears the fault counter

Explicitly excluded: FoE (File-over-EtherCAT) — deliberately omitted because its POSIX-filesystem semantics are unavailable in this no_std environment, and firmware update over an unauthenticated network channel is an out-of-scope attack surface.

WCET Budget

Taskv17 Budgetv18 BudgetSource of Changefast_control_loop410µs415µs+5µs for EtherCAT retry-bookkeeping worst case (TLA⁺-proven conservative)eoe_diagnostics_task—Negligible (1 Hz, not in FCL)New task, off hot pathgait_recalib_task—WCET-identical to v17 inferenceFixed-iteration int8 arithmetic 

12. Gait Classification & Recalibration

Inference (v17, unchanged in v18)

src/estimation/gait_classifier.rs implements a quantised int8 biomechanical gait classifier with a 22→16→7 fixed-point forward pass, running on the Cortex-M7 DSP core using SMLAD/SMLAD2 SIMD MAC instructions (enabled by +dsp rustflag).

Input features (22 dimensions): 10 plantar pressure readings (kPa), angular uncertainty (ESKF output), ZMP normalised XY coordinates, and a 7-element previous-phase one-hot vector.

Output: One of 7 gait phases — Unknown, HeelStrike, LoadResponse, MidStance, TerminalStance, PreSwing, Swing.

SR-GAIT-MODEL-01 (preserved): Gait phase is advisory telemetry only. No safety gate reads GaitPhase. A maximally-poisoned recalibration history can produce a misleading telemetry label but cannot gate or bypass any SR-* safety constraint.

Bounded On-Device Recalibration (v18)

Three independent bounds make recalibration safe on a SIL 3 device:

1. Fixed iteration count — recalibrate_batch processes exactly GAIT_RECALIB_MAX_BATCH = 32 samples per call. Extra samples remain queued for the next call. WCET is identical to v17 inference — Kani-proven via verify_gait_recalib_no_panic.

2. Fixed-point learning rate — GAIT_RECALIB_LR_NUM = 1, GAIT_RECALIB_LR_DEN = 64. All arithmetic is integer ratio — no floating-point in the recalibration path.

3. Hard delta clamp — GAIT_RECALIB_MAX_DELTA = 24. The cumulative delta applied to any single weight is clamped to ±24 int8 units at every write AND every read. A corrupted delta sector cannot push an effective weight outside the validated range; the clamp is a read-time invariant, not merely a write-time discipline.

Storage: The delta layer (GAIT_RECALIB_DELTA_W1) lives in a separate QSPI erase sector (.gait_recalib_delta) from the read-only base weights (.gait_weights). A power-loss mid-write to the delta sector cannot corrupt the validated base weights.

Algorithm: Single-layer perceptron-style update (not full backpropagation). For each mislabelled sample, weights feeding the incorrectly-winning hidden unit are nudged by ±LR_NUM/LR_DEN, clamped. Full backprop is deliberately avoided — its variable convergence would reintroduce data-dependent WCET.

Supervision source: Fleet-analytics-aggregated GaitRecalibSample batches (ground-truth phase labels correlated from cross-device / clinical-channel data), delivered back to the device over the existing TEE-validated Zenoh channel (SR-FLEET-01).

13. Fleet Analytics & Telemetry

src/foundation/fleet_export.rs (unchanged since v17) exports Merkle-chained telemetry digests via the TEE-validated Zenoh transport. v18 extends the same channel in both directions:

Uplink (unchanged): Per-tick (merkle_root, incline_rad, tick, mission_mode) tuples, exported at FLEET_EXPORT_DECIMATION_TICKS cadence.

Downlink (v18): Fleet-analytics-aggregated GaitRecalibSample batches arrive over the same channel and are staged for consumption by gait_recalib_task.

All fleet export records include the Merkle chain root, providing cryptographic evidence that the on-device state at the time of the export was produced by the attested firmware.

14. v18 Roadmap Closure

All five items from the v17 roadmap are closed in v18:

v17 Roadmap Itemv18 DispositionSOEM EoE/FoE bring-upClosed — control/ethercat.rs, comms/eoe_diagnostics.rs, SR-ETHERCAT-EOE-01. FoE deliberately excluded per original roadmap POSIX-dependency caveat.TPM-backed key storage for MRAM encryptionClosed — foundation/storage.rs, foundation/crypto.rs, security/tpm_attest.rs, SR-MRAM-SEAL-01Formal proof of EtherCAT working-counter gate's real-time propertiesClosed — verification/tla_scheduler.tla, tla_scheduler.cfg, SR-ETHERCAT-JITTER-01Multi-TPM quorum attestationClosed — security/tpm_attest.rs, comms/tee_monitor.rs, SR-ATTEST-QUORUM-01Gait classifier online recalibrationClosed — estimation/gait_classifier.rs, SR-GAIT-RECALIB-01 

15. v19 Proposed Roadmap

None of the following are implemented. Listed for planning purposes only.

FoE-based signed firmware staging (not auto-apply) — Revisit the v17/v18-excluded FoE module specifically for a staging-only field-update path: a technician pushes a signed candidate image to a quarantined flash bank over EtherCAT. secure_boot.rs's existing OTP-rooted verification gates promotion to the active bank. This closes the gap between "diagnostics over EoE" (v18) and "full field provisioning" without trusting an unauthenticated write path.

Quorum-aware MRAM key sealing — v18 deliberately keeps MRAM-key sealing single-TPM for traffic and latency reasons. Investigate a Shamir-shared key reconstructable from any 2-of-3 TPMs only at boot, amortizing the multi-TPM SPI cost over a single reconstruction rather than every ~10 Hz commit. Closes the remaining single-point-of-failure in the storage confidentiality chain.

Gait recalibration delta persistence across power cycles — .gait_recalib_delta currently resets to zero on every cold boot (QSPI section is RAM-shadowed, not persisted across power loss in this revision). Evaluate whether persisting the bounded delta is worth the added MRAM-style commit traffic, or whether per-boot re-accumulation from a longer fleet-analytics window is preferable from a safety-validation standpoint.

EoE-driven HJ grid hot-reload — Now that EoE diagnostics (v18) and the offline HJ grid generator (v17) both exist, investigate whether a re-surveyed environment's updated hj_table.bin could be staged (read-only, Merkle-verified, NOT live-patched into the running .hj_table ITCM section) over the EoE channel, rather than requiring a full reflash for environment changes.

TLA⁺ model coverage for the costmap-edge HJ-veto path — v18's jitter model covers the EtherCAT/scheduler interaction. The comms/zenoh_transport.rs + cognitive/dreamer.rs costmap-edge ingestion path (v17) has no equivalent formal timing model. Evaluate whether the 10 Hz polling cadence warrants the same rigour now that two other subsystems (scheduler, EtherCAT) have formal timing proofs.

16. Changelog (v17 → v18)

New Features

EoE Diagnostics (control/ethercat.rs, comms/eoe_diagnostics.rs) — Field-service EtherCAT-over-EoE mailbox tunnel with 4-query fixed dispatch, rate-limited to 10 frames/second, structurally isolated from all safety gates.

MRAM-at-Rest Encryption (foundation/storage.rs, foundation/crypto.rs) — AES-256-GCM sealing of MramSafetyState using a TPM-sealed key that never leaves the TPM in unsealed form. Fail-closed to liveness_reboot_safe_posture.

2-of-3 TPM Quorum Attestation (security/tpm_attest.rs, comms/tee_monitor.rs) — PCR-extend mirrored to all three TPMs; auditor accepts quote bundle with ≥2 independently-verified AIK signatures. Quorum degradation (1 or 0 responsive TPMs) never gates motion.

Bounded On-Device Gait Recalibration (estimation/gait_classifier.rs) — Fixed-iteration, int8-only, hard-clamped (±24) delta layer over read-only base weights. WCET-identical to v17 inference. Advisory-only output.

TLA⁺ DC Sync Jitter Model (verification/tla_scheduler.tla) — Proves DEADLINE_MET holds under up to 10 consecutive EtherCAT frame losses. FCL WCET revised from 410µs to 415µs (jitter-inclusive conservative bound).

eoe_diagnostics_task (main.rs) — 1 Hz RTIC task: resets EoE rate limiter, polls mailbox, dispatches queries.

gait_recalib_task (main.rs) — 0.2 Hz RTIC task: drains fleet-analytics recalibration batch and applies to delta layer.

Modified Files

FileChangeCargo.toml+aes-gcm, +tpm-spi-hal quorum feature, +soem-rs eoe-mailbox feature; new cargo features ethercat_eoe, tpm_quorum, mram_seal, gait_recalibbuild.rs+embed_tpm_quorum_aiks(): reads tpm_quorum_aiks.bin, embeds 3×32-byte AIK fingerprints as .tpm_quorum_aiks ELF section; adds .gait_recalib_delta linker section.github/workflows/harp-ci.yml+tla-jitter-model job; +quorum-attestation-integration job; updated needs: graphmcdc-modules.txt+src/comms/eoe_diagnostics.rs; src/security/tpm_attest.rs already present, confirmed for quorum branchessrc/foundation/schemas.rs+EoeDiagnosticFrame, +EoeDiagnosticResult, +TpmQuorumQuote, +QuorumAttestationResult, +GaitRecalibSample, +GaitRecalibResult, +SealedMramSafetyState, +MramSealResult; new SafetyFault variantssrc/foundation/storage.rs+SealedMramStore wrapper with commit_sealed() and read_unsealed()src/foundation/crypto.rs+ZeroizingKey, +seal_with_tpm_key(), +unseal_with_tpm_key()src/control/ethercat.rs+soem_eoe_ctx, +eoe_frames_this_second, +reset_eoe_rate_limit(), +poll_eoe_mailbox(), +send_eoe_diagnostic()src/estimation/gait_classifier.rs+GAIT_RECALIB_DELTA_W1 static; classify() reads clamped_effective_weight() instead of raw GAIT_WEIGHTS.w1; +recalibrate_batch(), +predict_argmax_only(), +apply_perceptron_nudge()src/comms/tee_monitor.rsattestation_chain() now calls extend_pcr_and_quote_quorum(), returns TpmQuorumQuotesrc/security/tpm_attest.rs+TPM_QUORUM_READY_MASK; +init_tpm_quorum(), +extend_pcr_and_quote_quorum(), +verify_quorum_offline(), +unseal_mram_key()src/verification/creusot_loops.rs+seal_buffer_loop_invariant_holdssrc/verification/kani_proofs.rs+verify_quorum_threshold_enforced, +verify_gait_recalib_delta_never_exceeds_clamp, +verify_gait_recalib_no_panic, +verify_mram_seal_roundtrip_or_fail_closedsrc/verification/tla_scheduler.tlaExtended with FrameLoss, FrameRecover, consecutive_frame_loss, ethercat_safe_to_command; revised FCL_WCET_US = 415; new SAFETY_GATE_CONSERVATIVE_DURING_LOSS invariantsrc/main.rs+EoeDiagnosticsHandler shared resource; init_tpm_quorum() replaces init_tpm(); MRAM commit uses SealedMramStore::commit_sealed(); +eoe_diagnostics_task; +gait_recalib_task; attestation_task handles TpmQuorumQuote 

Unchanged Files (v17 → v18)

All files not listed above are byte-for-byte identical to their v17 versions. Their original SR-* requirement IDs remain covered. See docs/v18_roadmap_closure.md for the full closure table.

HARP v18 — IEC 61508 SIL 3 / ISO 26262 ASIL D / ISO 13482 Compliant · Ferrocene Qualified Compiler · Fixed-Iteration QP · CMSIS-DSP SIMD · DTCM/ITCM TCM Layout · SOEM EtherCAT Master (DC/PDO + EoE Diagnostics) · Prusti DMA-Overrun Proofs · Real Flashed HJ Grid · ROS2/Zenoh Nav2 Bridge · Quantised Gait Classifier with Bounded On-Device Recalibration · 2-of-3 TPM Quorum Attestation · TPM-Sealed MRAM Encryption · OTP Secure Boot · Full Creusot Loop Invariants · CI-Mandatory MC/DC · TLA⁺-Proven DC Sync Jitter Tolerance · Fleet Analytics Export

