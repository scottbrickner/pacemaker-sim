# Enhancement Brief — Medtronic 5392 Temporary Pacemaker Simulator (Claude Code)
### Optimize and polish an existing, working single-file HTML simulator

This is an **enhancement** job, not a rebuild. The simulator already works and is clinically detailed. Your task is to raise the interface quality to a high polish, harden the facilitator/learner two-screen experience, and optimize the code — **without changing what makes the device faithful or breaking its clinical logic.**

---

## 1. Approach (decided)

- **Stay single-file HTML/CSS/JS.** No framework, no build step, no external runtime dependencies. The file must keep opening by double-click and running offline on any hospital machine or bedside tablet.
- Keep the **Medtronic 5392 faceplate photo-real** — refine polish only; do not restyle or "brand" the device itself.
- Bring vasoactive-sim-grade visual polish to the **surrounding surfaces** (facilitator dashboard, session/pairing UI, coaching panel, rhythm strip, typography, layout) — not the device.

## 2. What already exists (preserve all of this)

- **Device:** true 5392 proportions; canvas gauges for Rate, A Output (mA), V Output (mA); MENU parameter knob; Lock/Unlock key; Up/Down keys; DOO emergency button; power; LCD showing mode and lock state.
- **Full mode set:** DDD, DDI, VVI, DOO, VOO, AOO.
- **Clinical logic:** capture and sensing behavior, threshold testing, battery model, underlying-rhythm interaction, pause/self-test.
- **Connections / circuit model:** generator → 5492AL/5492VL patient cable → StreamLine epicardial heart wires → myocardium; tap points show green (made) / red (broken); a broken channel = no output and no sensing on that channel. Epicardial (post-op) vs transvenous setups. This is a core troubleshooting asset — keep it.
- **Rhythm strip:** `canvas#ecg` with colored atrial/ventricular pacing spikes and capture morphology.
- **Roles & sync:** `?role=learner` vs control (facilitator); `learnerLink()` builds a learner URL with relay + code; sync via BroadcastChannel (same-machine two windows) + localStorage fallback + optional WebSocket relay (two devices). `body.learnerview` hides facilitator panels.
- **Scenarios:** quick scenarios + CT-surgery stepped scenarios with coaching hints; facilitator channel-disconnect to create "why isn't it pacing?" cases.

## 3. Non-negotiable guardrails

- **Do not alter device fidelity.** Parameter ranges, mode behaviors, lock behavior, DOO emergency behavior, mA/mV steps must continue to match the real Medtronic 5392. Preserve the existing values; if you change any, verify against Medtronic 5392 documentation and flag it.
- **Do not break the clinical logic.** Capture, sensing, threshold testing, and the connections/circuit model must produce the same clinical outcomes after your changes as before. Treat these as behavior to protect, not refactor casually.
- **Keep it single-file and offline-openable.** No bundlers, no npm runtime deps, no CDN requirements for core function (a QR helper may be inlined). It must still work opened as a local file with no server.
- **Preserve the sync layer's capabilities** (same-machine and two-device). You may improve its UX and resilience, not remove either path.
- **Learner view must never expose facilitator tools.**

## 4. Enhancement roadmap (phases — checkpoint, preview, and commit after each)

**Phase 0 — Audit & plan (no edits).** Read the whole file. Produce: a short map of its structure (state model, render loop, device controls, strip, connections, roles/sync, scenarios); a list of the concrete polish/optimization targets you see; and the phased plan you propose. Wait for approval before editing. Commit the untouched baseline to git first.

**Phase 1 — Safe internal organization (no behavior change).** Within the single file, organize the CSS and JS into clearly commented sections; lift a small internal design-token block (colors, spacing, type) to the top so the surrounding-surface polish is consistent and easy to tune. Zero visual or behavioral change expected — this is groundwork. Verify parity.

**Phase 2 — Surrounding-surface polish (device untouched).** Raise typography, spacing, and hierarchy to vasoactive-sim quality on everything that is *not* the device: the facilitator dashboard, the session/pairing UI, the coaching/hint panel, the scenario controls, and the rhythm-strip framing. Micro-refine the device only (e.g., crisper shadows/labels) without changing its realism.

**Phase 3 — Two-screen experience & flexible deployment.** Make starting a synced session obvious and resilient for BOTH setups: (a) two windows on one machine (BroadcastChannel, zero server) as the default one-click path; (b) two separate devices via the relay, with a clean pairing flow — show the learner link as a scannable **QR code** plus a short code, a clear connection/latency indicator, and automatic reconnect. Make the mode choice explicit in the UI so a facilitator isn't guessing.

**Phase 4 — Facilitator power tools.** A tidy fault-injection library with one-click, clinically-named events: loss of capture (raise threshold above output), failure to sense / undersensing, oversensing & crosstalk, lead/channel disconnection, battery depletion, and underlying-rhythm/competition changes. Fold the existing quick + CT-stepped scenarios into a single clean scenario library with stepped progression and facilitator notes. Nothing here should be reachable from the learner view.

**Phase 5 — Debrief & event log.** Record a timestamped timeline of learner device changes, injected faults, and resulting capture/sense outcomes; present it as a facilitator-side debrief and allow export (copy/download). This mirrors the vasoactive sim's debrief value and is the highest-leverage teaching add.

**Phase 6 — Accessibility & responsiveness.** Add responsive layout so the learner view works full-screen on a projector and on a tablet (touch targets ≥ 44px); keyboard operability and ARIA labels for the controls and canvas gauges; and confirm the canvas render loop uses requestAnimationFrame efficiently with clean teardown. No console errors or warnings.

## 5. Acceptance criteria (verify each)

- The 5392 faceplate looks materially the same as the original (photo-real preserved); only surrounding surfaces are visibly upgraded.
- Capture, sensing, threshold testing, and the connections model produce identical clinical outcomes before vs. after.
- Two windows on one machine sync with **no server**; two separate devices sync via the relay using a QR/code pairing flow; both survive a disconnect/reconnect.
- The learner view exposes zero facilitator controls.
- The facilitator can inject each named fault in one click and see it reflected on the learner display and strip.
- A debrief event log is produced and exportable.
- Learner view is usable on a tablet and a projector; keyboard and screen-reader labels are present; console is clean.
- The file still opens and runs as a local file with no build step.

## 6. Fidelity note

Keep the device true to the real Medtronic 5392, including the accurate cabling/wire references already in the file (5492AL/5492VL patient cable, StreamLine epicardial wires). Do not invent device behaviors; when unsure, preserve the existing behavior and flag the question rather than guessing.
