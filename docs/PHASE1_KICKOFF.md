# Kickoff — paste into the Claude Code **Code** tab (in Plan mode)

Start with an audit, not edits. This message runs Phase 0.

```
You're enhancing an existing, working single-file simulator: 5392-pacemaker-simulator.html
(a Medtronic 5392 temporary pacemaker). This is an ENHANCEMENT, not a rebuild.

First read CLAUDE.md and docs/ENHANCEMENT_BRIEF.md in full, then read the simulator file.

This session is PHASE 0 (Audit & plan) — do NOT edit anything yet:
1. Commit the untouched file to git as the baseline (git init if needed).
2. Give me a short map of the file's structure: state model, render/animation loop,
   device controls, the ECG strip, the connections/circuit model, the roles/sync layer,
   and the scenarios.
3. List the concrete polish and optimization targets you see, grouped by the phases in the
   brief (surrounding-surface polish, two-screen/pairing UX, facilitator power tools,
   debrief/event log, accessibility/responsiveness, code organization).
4. Propose your phased plan and flag anything risky to the clinical logic or sync.

Hard constraints (from CLAUDE.md): keep it single-file and offline-openable; keep the 5392
faceplate photo-real (polish surrounding surfaces only); do not break capture/sensing/
threshold/connections behavior; preserve BOTH sync paths (same-machine and two-device);
learner view must never expose facilitator tools.

Wait for my approval before starting Phase 1.
```

## After approval, run phases one at a time, e.g.:
```
Approved. Proceed to PHASE 1 (safe internal organization, no behavior change) from
docs/ENHANCEMENT_BRIEF.md. Keep the device and all clinical outcomes identical; this phase is
groundwork only. Show me the file running in preview and confirm parity, then stop.
```
Work one phase per session and commit after each. Test the learner view by opening the file with `?role=learner` appended to the URL.
