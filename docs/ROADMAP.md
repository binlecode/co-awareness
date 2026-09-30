# ROADMAP

The product tracker: candidate backlog (`R<n>`). Shipped work, architectural invariants, and subsystem boundaries (including declined proposals and re-open triggers) are documented as-built in `docs/ARCHITECTURE.md`.

Items are `R<n>`, assigned once, never reused. **P1** user-visible defect or silent failure · **P2**
real capability gap · **P3** nice to have · **P4** parity for its own sake. Nothing here is a
commitment or a date.

Keep this at tracking altitude — item, priority, blocker, and any catch an implementer would trip
over. Design and technical rationale live in `docs/ARCHITECTURE.md`; release history lives in `CHANGELOG.md`
and git. Refer to code by **symbol, never by line number** — anchors rot every release.

---

## Open

Ordered by ROI, highest first — value against cost, not just the priority band. Each row is one line of
tracking: what is wrong, and where the design lives. Mechanism, schema, parameters and verification live
in the design the row points to — never restated here.

| ID | Item | Pri | Blocked by |
|---|---|---|---|
| R29 | **Retire the legacy-argv bridge.** Delete the launcher's `legacy_argv`, `Restarter.migrateLegacyLoginItem`, and qa.sh §6's legacy `--precompile` row together; bridge rationale in `docs/ARCHITECTURE.md` § 2. | P3 | Installed copies have been through one launch on the verb-first grammar (which rewrites their login item); removing it earlier strands an un-updated install's restart and login start |
