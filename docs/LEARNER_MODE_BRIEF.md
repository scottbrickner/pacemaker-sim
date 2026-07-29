# Build Brief — Self-Directed Learner Practice Mode (Medtronic 5392 Simulator)
### A standalone, offline practice experience for RNs: threshold trainer + troubleshooting randomizer, with a background attempts log that matches the vasoactive/ZOLL platform

This is an **additive enhancement** to `5392-pacemaker-simulator.html`. It adds a self-directed learner mode — no facilitator, no second screen — where an RN practices pacing thresholds and pacemaker troubleshooting against randomized, unique cases, gets either inline coaching or a self-assessment debrief, and has every attempt logged in the background exactly the way the vasoactive and ZOLL sims log theirs.

Everything here obeys the existing `CLAUDE.md` guardrails: **single file, no build step, offline-openable, device fidelity and clinical logic untouched, learner view exposes zero facilitator tools.** The practice mode reuses the engine that already exists (capture/sense/threshold/connections/rhythm) and the 12 authored **PACE** cases.

---

## 0. How this fits the platform

- **Reuse, don't rebuild.** The device, the rhythm/capture/sense engine, the connections model, the ECG strip, and the PACE scenario data already exist. Practice mode is a new *shell* around them.
- **Match the platform's skill-tracking pattern.** The vasoactive and ZOLL sims share an attempts-log design (`engine/skillAttempt.ts`, `state/skillTrackingStore.ts`, `sync/skillTracking.ts`, `sync/skillTrackingWorkbook.ts`, `sync/teamsFolder.ts`, `lib/learnerIdentity.ts`). Port that pattern to vanilla JS here so a learner's record is consistent across all three sims and lands in the same Teams channel.
- **Two feedback modes = the platform's two run modes.** The platform already uses `mode: 'training' | 'validation'`. Map it 1:1:
  - **Inline coaching → `training`** (real-time nudges + end debrief; unlimited hints).
  - **Debrief-only self-assessment → `validation`** (silent during the run; scorecard + best-practice debrief at the end; only a validation pass counts toward a skill sign-off).

---

## 1. Guardrails (inherit + add)

Inherit all of `CLAUDE.md` and the `ENHANCEMENT_BRIEF.md` non-negotiables. Add:

- **New standalone role `?role=practice`** (a third role beside control/facilitator and `?role=learner`). `body.practiceview` shows only the device, the strip, and the practice shell — **no facilitator panel, no session/pairing/relay UI**. Practice mode never opens a sync channel.
- **No new device behaviors.** Practice mode only *reads* the engine's ground truth (thresholds, signals, faults, capture/sense outcomes) and *sets* the same scenario fields the PACE data already uses (`rhythm, aRate, vRate, aThr, vThr, aSig, vSig, leadA/leadV, oversA/oversV, batt, connectionFault`). Randomization writes those fields; it does not invent new ones.
- **Offline-first.** Everything works opened as a local file. The attempts log degrades gracefully when storage or the File System Access API is unavailable (see §6).

---

## 2. New surfaces (the practice shell)

A learner entering `?role=practice` sees a **Practice Home**:

1. **Pick a module:** Threshold Trainer · Troubleshooter.
2. **Pick a mode:** Inline Coaching (`training`) · Self-Assessment (`validation`).
3. **Optional filter:** Level of care — All · Telemetry · Stepdown · ICU/CTICU (drives which PACE cases / difficulty band is drawn).
4. **Learner identity** (name + `@med.usc.edu` email): required before a `validation` run (so the attempt can be signed off); optional for `training`. Reuse the `isValidInstitutionalEmail` check.
5. Running **scorecard** during/after; **printable self-assessment** at the end (§7).

Keep the practice shell in the "surrounding surfaces" visual register (polished app chrome), never restyling the device.

---

## 3. Module A — Threshold Trainer (randomized, unique each run)

Practice the two threshold procedures the real device teaches, on both channels, with the target values **hidden and randomized** each round so no two runs are the same.

### 3.1 Drills
- **Ventricular stimulation (capture) threshold**
- **Atrial stimulation (capture) threshold**
- **Ventricular sensing threshold**
- **Atrial sensing threshold**
(Random draw, or learner-chosen. A round can chain "find threshold → set safe margin.")

### 3.2 Randomization model (clinically realistic, weighted)
Draw hidden values from weighted bands so common cases dominate and teaching-edge cases appear occasionally. All within engine ranges (`aThr 0.1–20, vThr 0.1–25, aSig 0.2–10, vSig 0.5–20`). Exact bands/weights live in `learner-practice-data.json → thresholdTrainer.randomization`.

- Capture threshold: mostly 0.5–2.5 mA (good wire), sometimes 2.5–5, occasionally 5–10 (reposition-territory), rarely >10 (bad wire — learner should recognize and flag).
- Sensed amplitude: P-wave mostly 1.5–4 mV, R-wave mostly 5–14 mV; occasional small signals (P <1, R <4) to make sensing matter.
- Each round is wrapped in a randomly chosen **clinical frame** (e.g., "POD 2, verify V capture before weaning"), pulled from `thresholdTrainer.scenarioFrames`.

### 3.3 Task flow (mirrors the Medtronic procedure the engine already models)
- **Stimulation threshold:** set rate above intrinsic → lower output until loss of capture → raise until consistent 1:1 capture (= threshold) → **set output to 2× threshold (double)**. *Local training standard is 2× (double), not 3× — a tighter margin unmasks a rising threshold / dislodgement sooner and spares the battery. The manufacturer permits 2–3×; coach a learner who cranks to 3×+ back toward doubling.*
- **Sensing threshold:** ensure non-pacing (rate below intrinsic, output minimal) → adjust sensitivity to the least-sensitive mV that still senses (= the signal amplitude) → **set sensitivity to ≤ ½ of that** (lower mV = more sensitive).

### 3.4 What the trainer detects (auto-grading)
Read the engine's hidden value and the learner's final settings:
- Threshold **identified** within tolerance of the hidden value (default ±0.5 mA capture / ±1 sensitivity step — see `tolerances`).
- **Safe margin** set correctly (output at 2× threshold — double, the trained standard; sensitivity ≤ ½ signal).
- **Procedure** correct (rate set appropriately before testing; correct channel).
- **Safety** (didn't strand a dependent patient without capture; didn't set sensitivity so high it undersenses; recognized/flagged a high or "bad-wire" threshold).

### 3.5 Feedback
- **Inline (`training`):** live nudges — "output is below threshold, you'll drop capture," "you've found loss of capture — come up until every spike captures," "good, now set 2× for margin (double, not triple)," "lower the mV number to sense that small R wave."
- **Debrief (`validation`):** silent; end scorecard vs. best practice with the exact target values revealed.

Rubric categories in `rubrics.threshold` (see data file), shaped as platform `ScoreCategory[]` (`key,label,status,detail`).

---

## 4. Module B — Troubleshooter (curated + randomized, hybrid response)

Presents a patient situation (rhythm ± fault + context) and asks the learner to **recognize** it and **act**. Response is **hybrid** (your choice):
- **Judgment/recognition/escalation → multiple choice** (auto-graded).
- **Psychomotor actions → operate the real device** (increase output to regain capture, adjust sensitivity, press DOO for async, reconnect a cable, find a threshold) — the sim auto-detects the correct action via the engine (capture restored / appropriate sensing / async engaged / circuit closed).

A case is a short **sequence of decision points**, each either MCQ or device-action.

### 4.1 Case sources (both)
- **Curated:** the 12 **PACE** cases (by `key`), including the post-op asystole "check underlying rhythm" case, drawn in random order, with their hidden thresholds re-randomized within band so even a familiar case plays differently.
- **Randomized generator:** procedurally combine a **rhythm × fault × LOC context** into a fresh case. Only clinically coherent combinations are allowed — governed by `generator.validityMatrix` (which faults make teaching sense on which rhythms/contexts, with weights). This yields near-infinite unique reps without nonsense cases.

### 4.2 Fault → correct-action mapping (the grading brain)
For each fault the sim already models, `faultActionMap` defines: the correct **recognition** label, plausible **distractors**, the correct **action(s)**, how the engine **detects** a correct device action, and the **safety** items. Summary:

| Fault | Recognize | Correct action(s) | Engine detects |
|---|---|---|---|
| Loss of capture (spikes, no QRS; threshold > output) | "Loss of capture" | ↑ output to 2× threshold (double); ensure backup if dependent | capture restored (output ≥ threshold) |
| Lead fracture (no spikes) | "Lead fracture — not learner-fixable" | escalate/replace, transcutaneous bridge (↑output & async will NOT help) | recognizes non-fixable; no output path |
| Cable disconnect (no spikes; `connectionFault`) | "Disconnected cable at generator" | reconnect at the generator port (connection modal) | circuit closed → output resumes |
| Undersensing (spikes into intrinsic; signal < sensitivity setting) | "Undersensing / competition" | ↑ sensitivity (lower mV) to ~½ signal | sensing appropriate; competition stops |
| Oversensing (pauses, no spikes when needed) | "Oversensing" | ↓ sensitivity (raise mV) or go async; treat source | consistent pacing resumes |
| Dependence, not pacing (asystole / CHB vRate 0, capped) | "Needs pacing now" | initiate pacing / escalate / transcutaneous bridge | pacing initiated with capture |
| Battery low/depleted | "Battery failing" | replace battery/generator without interrupting a dependent patient | pacing maintained |
| Atrial flutter (advanced) | "Atrial flutter" | confirm order + defib on → deliver RAP above atrial rate | RAP delivered / conversion |
| Check underlying rhythm — post-op **asystole**, dependent | "Assess underlying rhythm safely" | step **rate down gradually** or brief **Pause**, monitored, backup ready — **never unplug the bridging cables** | rate reduced / Pause used (NOT a cable disconnect); pacing restored |

**Escalation is a first-class rec (per `escalationGuidance`).** When the learner has exhausted device + wire troubleshooting — **output at maximum without capture**, threshold >10 mA / climbing, **sensitivity at its limit** with a persistent sensing problem, or oversensing that can't be rescued — the sim surfaces (inline and in debrief): *notify the provider/EP, and plan alternative pacing — transcutaneous now as a bridge, anticipate transvenous or new epicardial wires.* This is the "don't keep cranking" lesson: at the device's limits with an unresolved problem, escalation + an alternative-pacing plan is the correct next step, not further adjustment.

### 4.3 Feedback
Same two modes. Inline gives per-decision feedback and a hint ladder; validation stays silent and scores at the end. Rubric categories in `rubrics.troubleshoot` (recognition, priority action, execution-on-device, safety), platform `ScoreCategory[]` shape.

---

## 5. Attempts log & scoring (match the platform exactly)

Port the vasoactive/ZOLL skill-tracking subsystem to vanilla JS. Same shapes, same storage discipline, same Teams handoff — so NPD gets one consistent record across all three sims.

### 5.1 `AttemptRecord` (identical shape, new `recordType`)
```
{
  recordType: 'pacemaker-skill-attempt',
  attemptId, module: 'threshold'|'troubleshoot', caseId, caseLabel,
  mode: 'training'|'validation',
  learnerName, learnerEmail,
  overallPercent, categories: ScoreCategory[], passed, recordedAt
}
```
- Recorded at **every** debrief (training and validation) so usage can be counted; `passed` is `mode==='validation' && isSkillPassed(card)`.
- `ScoreCategory` = `{ key, label, status: 'pass'|'partial'|'fail', detail }`.

### 5.2 Storage (separate, self-owned)
- Hand-rolled localStorage, **separate key** `'pacemaker-sim:skill-tracking'` (never shared with any sync/state key), try/catch safe, degrades to in-memory.
- `learnerIdentity` (name + `@med.usc.edu`) persisted alongside the attempts array.

### 5.3 Export & Teams handoff (port `teamsFolder.js`)
- **Per-attempt JSON + CSV** (one CSV row per scored category) — pure functions + a DOM `download()`.
- **Shared append-only workbook** saved into a **Microsoft Teams channel's Files folder** via the File System Access API: pick the folder once (user gesture), remember the handle in IndexedDB, silent appends thereafter; Chromium-only, fall back to download in Safari/Firefox. Read existing bytes → append one row → write back.
- **Optional telemetry hook** (`sendAttemptTelemetry`) — env/flag-gated Power Automate/Dataverse POST, off by default (offline-safe).

> **⚠ One build decision — single-file vs. the .xlsx workbook.** The platform's shared workbook uses SheetJS (`xlsx`), an npm dependency the pacemaker's single-file/offline rule forbids by default. Two options:
> - **(Recommended) Vendor an inlined SheetJS** as a same-file helper (the CLAUDE.md "inlined QR helper is acceptable" exception extends cleanly to one vendored library), so the pacemaker writes to the **same `.xlsx` workbook** as the other sims — one tracking file for NPD. Bloats the HTML but keeps it single-file/offline.
> - **(Purist) CSV-append instead of xlsx** — the pacemaker appends rows to a `Pacemaker-Sim-Skill-Tracking.csv` in the Teams folder (zero dependencies), and NPD opens/joins it in Excel. Keeps the file lean; costs the single-workbook consistency.
> Pick one; the JSON/CSV per-attempt export and the Teams-folder plumbing are dependency-free either way.

---

## 6. Printable self-assessment

At debrief, a **Print / Save-as-PDF** action renders a clean one-page summary: learner + timestamp, module/case, mode, overall %, each scored category with pass/partial/fail + detail, the best-practice targets (revealed), and next-step focus. A `@media print` stylesheet hides the device/shell and shows only the summary. This is the learner's keep-or-hand-in artifact (works even where the File System Access API doesn't).

---

## 7. Deployment — 5 West Teams, Knowledge Hub channel

Target: the practice sim lives as a **resource in the 5 West Team's Knowledge Hub channel**, and attempts flow into that channel's Files.

1. **Host the practice build.** Same pattern you already use (Netlify or the SharePoint site behind the Team). `?role=practice` (or a thin `practice.html` that sets the role) is the learner entry URL. It still works fully offline if opened locally.
2. **Surface it in the channel.** Add a **Tab** in the Knowledge Hub channel: a **Website tab** (or SharePoint page/link) pointing at the hosted URL. A full canvas sim doesn't embed inside a Power App, so this is the clean path; if the Knowledge Hub Power App is itself pinned as a tab, add the sim as a sibling resource tab/link, not an embed.
3. **Attempts → the channel's Files.** The Teams-folder save (§5.3) targets the **Knowledge Hub channel's Files folder** (which is a SharePoint document library synced by the OneDrive/Teams desktop client). The learner picks that synced folder once; the workbook then accumulates there for NPD. A QR/short link makes bedside/tablet access easy.
4. **Optional later — Dataverse.** If you want completions in the 5 West Knowledge Hub Power App's data model, enable the telemetry hook to POST each `AttemptRecord` to a **Power Automate** flow that writes to **Dataverse** (a `SkillAttempt` table). *Gotcha:* an offline single-file sim can't write Dataverse directly (no auth, breaks offline) — the flow/webhook is the bridge, and it only fires when online with the flag on. Until then, the Teams workbook is the system of record.

---

## 8. Phased roadmap (audit-first, preview + commit after each — same discipline as the enhancement brief)

- **P0 — Audit & plan (no edits).** Map where the engine exposes threshold/signal ground truth, capture/sense outcomes, fault state, and the PACE loader. Confirm the cleanest read-only hooks for practice mode. Propose the plan; commit baseline first.
- **P1 — Practice role & shell scaffolding.** Add `?role=practice` + `body.practiceview` (hides all facilitator/sync UI), Practice Home (module/mode/LOC/identity), and empty scorecard/debrief containers. No scoring yet. Verify learner/facilitator modes are untouched.
- **P2 — Threshold Trainer.** Randomization model, task flow, auto-detection, rubric, both feedback modes. Reuse the existing threshold engine — do not fork clinical logic.
- **P3 — Troubleshooter (curated).** Run the 12 PACE cases as decision-point sequences with the hybrid response and `faultActionMap` grading; both feedback modes.
- **P4 — Troubleshooter (generator).** Add the `validityMatrix` random generator for endless unique cases; log what it drops so coverage is honest.
- **P5 — Attempts log & scoring.** Port `skillAttempt`/`skillTrackingStore`/`skillTracking`/identity to vanilla JS; JSON/CSV export; separate localStorage key; printable self-assessment. Land the build decision from §5.3.
- **P6 — Teams handoff.** Port `teamsFolder.js`; folder-pick/remember/append; download fallback; (optional) telemetry hook wiring behind a flag.
- **P7 — A11y, responsiveness, deploy.** Tablet/projector, ≥44px targets, keyboard + ARIA on new controls, clean console; publish the hosted build and wire the Knowledge Hub channel tab + Files folder.

## 9. Acceptance criteria

- `?role=practice` shows device + strip + practice shell only — **zero facilitator tools, no sync UI**; existing facilitator and `?role=learner` behavior unchanged.
- Threshold Trainer produces a **different hidden target every run**; correctly grades threshold identification, safe margin, procedure, and safety; both feedback modes behave (training coaches live; validation stays silent until debrief).
- Troubleshooter runs the 12 PACE cases **and** generates valid random cases; hybrid response works (MCQ + device-detected actions); grading matches `faultActionMap`.
- Every completed run writes an `AttemptRecord` to the **separate** localStorage store; `passed` only on validation + pass; JSON/CSV export works; the Teams-folder workbook (or CSV) accumulates rows and a download fallback exists.
- Printable self-assessment renders a clean one-page debrief.
- Still opens and runs as a **single local file, offline**; device fidelity and capture/sense/threshold/connections outcomes are byte-for-byte unchanged; console clean.

## 10. Kickoff prompt (paste into Claude Code, Plan mode)

> Read `CLAUDE.md`, `docs/ENHANCEMENT_BRIEF.md`, and `docs/LEARNER_MODE_BRIEF.md`. Do **Phase 0 only**: audit the existing single file and report (a) where the engine exposes threshold/signal ground truth, capture/sense outcomes, fault flags, connection state, and the PACE scenario loader; (b) the read-only hooks practice mode should use so it never forks clinical logic; (c) the exact insertion points for a `?role=practice` shell that hides all facilitator/sync UI; and (d) your proposed phased plan per the brief. Make **no edits** and commit the untouched baseline first. Load `learner-practice-data.json` as the data contract for the trainer, generator, MCQ bank, and rubrics. Wait for approval before Phase 1.
