# PLAN: STARE_Viaticum Live Development Closed Loop
<!-- /autoplan restore point: will be written after intake -->

**Objective:** Eliminate all manual steps between lore intent and live UE5 execution by
wiring `/stare-sync` (lore layer) → `/ue5-stare` (constraint layer) → `/stare-ue5-mcp`
(execution layer) into a continuous, self-correcting development loop.

**Owner:** intermatrixnaut  
**Engine:** UE 5.8.1 · macOS M1 · Claude Sonnet 4.6  
**Status:** Active — gauntlet system just completed, loop integration is next.

---

## Dream State

Claude Code operates as a live co-developer inside STARE_Viaticum without any manual
script hand-off. When a new lore element is canonized in `/stare-sync`, its mechanical
representation (GAS attribute, DataTable row, Blueprint property) is verified against
`/ue5-stare` API rules and then written or patched directly in the running editor via
`/stare-ue5-mcp` tools — in a single session, no restarts, no copy-paste.

The three skills are **not three separate tools**. They are one loop:

```
Lore change (stare-sync) 
  → validate against safe API patterns (ue5-stare)
    → execute in live editor (stare-ue5-mcp)
      → verify result → update lore if behavior differs from intent
```

---

## Phase 1 — stare-ue5-mcp: Live RC Tooling (current)

### Goals
- [ ] MCP server registered in `~/.claude.json` and connects on session start
- [ ] All 15 tools verified against live editor (search_assets, set_property, run_python_in_editor, etc.)
- [ ] `is_editor_alive()` gate enforced before every in-editor write
- [ ] Aura port-file detection working (`rc_server_port.txt` → fallback 30010)
- [ ] Gauntlet system validated: DT_GauntletTiers_VhrexKaulvoids imported, component on BP

### Constraints
- Must use UE5's bundled Python 3.11 (pydantic_core ABI — system Python 3.14 fails)
- RC class names must be fully qualified: `/Script/Engine.DataTable` not `DataTable`
- Port is dynamic — never hardcode; always read from Aura's port file first

### NOT in scope
- Writing to version-controlled assets via RC (use `-ExecutePythonScript` for batch jobs)
- Multiplayer or replication changes via RC

### What already exists
- `Tools/stare_mcp_server.py` — FastMCP server with 15 tools
- `~/.claude/skills/stare-ue5-mcp/SKILL.md` — slash command + registration steps
- `~/.claude.json` — stare-ue5 registered (needs Claude Code restart to activate)

---

## Phase 2 — ue5-stare: Harness Skill Maintenance

### Goals
- [ ] Harness skill covers all current C++ modules: STARE, STARECreatures, STAREMusicDriver, STAREMusicDriverEditor
- [ ] Python API table in skill is current — every `unreal.*` symbol verified against UE5 stubs
- [ ] Build.sh pattern documented: `-Target=STAREEditor -Platform=Mac -Configuration=Development`
- [ ] Known error registry updated with: pydantic_core ABI, FTopLevelAssetPath fully-qualified requirement, WidgetTree Python-blocked
- [ ] Pre-flight checklist embedded: RESTART editor after any new USTRUCT/UENUM before running Python

### Constraints
- No MCP/RC code until `pgrep -x UnrealEditor` returns 0
- WidgetTree child widgets: C++ NativeConstruct only — never BindWidget+manual UMG
- UE5 Python strips leading E from UENUM: `ESTAREFeatCategory` → `unreal.STAREFeatCategory`

### NOT in scope
- Third-party plugin modifications
- Engine source changes

### What already exists
- `~/.claude/skills/ue5-stare/SKILL.md` — full harness, build commands, API table, error fixes

---

## Phase 3 — stare-sync: Lore Layer → UE5 Pipeline

### Goals
- [ ] `/stare-sync` runs clean: inventory → extract → 11 files → reconcile → index → PDF
- [ ] character_archives.json includes all characters built this session: Slillith, Lhellia, Ihzuuluvia, Hayzelle, Ponse, + new archetype characters from Matrix/Terminator mapping
- [ ] vehicle_fleet_registry.json includes VhrexKaulvoids gauntlet as gear entry, not vehicle
- [ ] psi_sync_matrix includes VNA node mappings (7 nodes, T1/T2/T3 assignments)
- [ ] STARE_Universe_Codex.pdf regenerated after new archetype lore is added
- [ ] Cross-refs wired: gauntlet system ↔ VNA nodes ↔ Crown Torque canon doc

### Constraints
- S.T.A.R.E. and Neo-Egyptian mythology never conflated in the same cross-ref
- character_archives entries require: source_files[], cross_refs[], BIR, shard, arc_status
- Songs need SUNO_AI_SPECIFICATIONS_MANDATORY.md compliance flag

### NOT in scope
- Notion population (separate plan: PLAN-stare-notion-character-population.md)

### What already exists
- `~/.claude/skills/stare-sync/SKILL.md` — full pipeline
- `docs/lore/` — 30+ source markdown files
- `docs/lore/source/CHARACTER _ VEHICLE.pdf` — name registry

---

## Phase 4 — Closed Loop Integration

### Goals
- [ ] Documented workflow: lore change → which skill to invoke → expected output
- [ ] Session startup checklist: `pgrep UnrealEditor` → load stare-sync memory → verify MCP live
- [ ] `/stare-sync add <name>` wired to trigger MCP DataTable row append (stretch)
- [ ] Gauntlet system end-to-end validated: CSV → UE5 DataTable → BP component → tier promotion works in PIE

### The loop, step by step
1. **New lore** — user canonizes a character, vehicle, or mechanic
2. **stare-sync add** — updates character_archives + cross_refs, flags any SUNO violations
3. **ue5-stare preflight** — verify API symbols, check editor gate
4. **stare-ue5-mcp execute** — create/update asset in live editor (DataTable row, GE, BP property)
5. **Verify in PIE** — test the behavior, compare against lore intent
6. **stare-sync reconcile** — if behavior differs from intent, note in timeline_integrity_log

---

## Failure Modes

| Mode | Symptom | Recovery |
|------|---------|----------|
| Editor closed during MCP call | HTTP 503 / connection refused | Check gate, open editor, retry |
| stale port file | Wrong port, RC calls fail | Delete `rc_server_port.txt`, reopen editor via Aura |
| New USTRUCT not reflected | AttributeError on unreal.* | Save all, restart editor, re-run script |
| pydantic_core ABI mismatch | ModuleNotFoundError on MCP start | Confirm command path = UE5's Python 3.11 |
| stare-sync conflict | File on disk ≠ seed canon | File always wins — log in timeline_integrity_log |
| DataTable row struct mismatch | CSV import fails to find row struct | Recompile project, restart editor, reimport |

---

## Error & Rescue Registry

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `Ensure condition failed: false` in FRCAssetFilter | Bare class name in RC filter | Use `/Script/Module.ClassName` format |
| `ModuleNotFoundError: pydantic_core._pydantic_core` | System Python 3.14 vs ABI | Use `/Users/Shared/Epic Games/UE_5.8/.../Python3/Mac/bin/python3` |
| `EXxx not found` in Python | UE5 strips E prefix from UENUM | `unreal.XXx` not `unreal.EXxx` |
| `WidgetTree is PROTECTED` | Python trying to set UMG children | Build widget hierarchy in C++ NativeConstruct |
| Build.sh `Result: Failed` after new USTRUCT | Hot-reload won't register reflection | Restart editor after any new USTRUCT/UENUM |

---

## Success Criteria

- [x] MCP server installed and skill registered
- [x] STAREGauntletTypes.h + STAREGauntletComponent.h/cpp written
- [x] DT_GauntletTiers_VhrexKaulvoids.csv with all 5 tiers
- [x] setup_gauntlet_component.py automates import + Blueprint wiring
- [ ] MCP verified live in a new session (requires Claude Code restart)
- [ ] Gauntlet tier promotion tested in PIE with a Chronotrooper BP
- [ ] stare-sync run after Matrix/Terminator archetype characters added
- [ ] ue5-stare skill updated with STAREGauntletComponent API entries

---

---

## /autoplan Review Report

### CEO DUAL VOICES — CONSENSUS TABLE [subagent-only]
```
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Premises valid?                   ISSUE    N/A   FLAGGED — assumed lore=bottleneck
  2. Right problem to solve?           ISSUE    N/A   FLAGGED — no PIE validation gate
  3. Scope calibration correct?        OK       N/A   N/A
  4. Alternatives sufficiently explored?ISSUE   N/A   FLAGGED — no version-lock strategy
  5. Competitive/market risks covered? MEDIUM   N/A   N/A
  6. 6-month trajectory sound?         ISSUE    N/A   FLAGGED — automate bad design risk
═══════════════════════════════════════════════════════════════
```

### DX DUAL VOICES — CONSENSUS TABLE [subagent-only]
```
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Getting started < 5 min?          CRIT    N/A   CRITICAL — no cold-start path
  2. API/CLI naming guessable?         ISSUE   N/A   FLAGGED — slash vs bare name
  3. Error messages actionable?        CRIT    N/A   CRITICAL — no timeout/hang path
  4. Docs findable & complete?         ISSUE   N/A   FLAGGED — zero copy-paste examples
  5. Upgrade path safe?                N/A     N/A   N/A
  6. Dev environment friction-free?    ISSUE   N/A   FLAGGED — no port override escape hatch
═══════════════════════════════════════════════════════════════
```

### ENG DUAL VOICES — CONSENSUS TABLE [subagent-only]
```
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Architecture sound?               ISSUE   N/A   FLAGGED — no schema contract
  2. Test coverage sufficient?         ISSUE   N/A   FLAGGED — 3 PIE cases missing
  3. Performance risks addressed?      OK      N/A   N/A
  4. Security threats covered?         CRIT    N/A   CRITICAL — RC HTTP unauthenticated
  5. Error paths handled?              ISSUE   N/A   FLAGGED — port-file race + no retry
  6. Deployment risk manageable?       ISSUE   N/A   FLAGGED — hot-reload crash untested
═══════════════════════════════════════════════════════════════
```

### Cross-Phase Themes
**Theme 1: No inter-layer validation** — CEO + DX + Eng all independently flagged the
absence of a contract between stare-sync output and MCP/UE5 struct fields. High-confidence signal.

**Theme 2: Error handling gaps** — CEO (RC fallback missing) + DX (timeout/hang missing)
+ Eng (ConnectionRefused race, port-file race). The error table exists but is incomplete.

### Architecture Diagram
```
STARE_Viaticum Live Development Loop
─────────────────────────────────────────────────────────────────────────────

  [ Lore / stare-sync ]                    markdown / JSON files
        │                                  character_archives.json
        │  stare-sync add <name>            vehicle_fleet_registry.json
        │  (Step 2: Extract & normalize)    psi_sync_matrix.json
        ▼                                        │
  ┌─────────────────────┐              ┌─────────────────────────┐
  │  stare-sync memory  │ ── reads ──► │  ue5-stare harness      │
  │  11 canonical files │              │  (API safety layer)     │
  └─────────────────────┘              │  - verified unreal.* API│
        │                              │  - build commands       │
        │ ←── schema validation ───    │  - error registry       │
        │      [MISSING → must add]    └─────────────────────────┘
        │                                        │
        ▼                                        ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  stare-ue5-mcp (FastMCP stdio, UE5 Python 3.11)            │
  │  stare_mcp_server.py → 15 RC tools                         │
  │  Port: reads ~/Library/.../rc_server_port.txt              │
  │  Auth: ⚠️  NONE (RC HTTP unauthenticated on localhost)     │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼  HTTP to 127.0.0.1:{dynamic port}
  ┌─────────────────────────────────────────────────────────────┐
  │  Unreal Editor 5.8.1 (Remote Control Plugin)               │
  │  DataTable import, Blueprint property set, Python exec     │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼  PIE validation
  ┌─────────────────────────────────────────────────────────────┐
  │  Human confirms mechanic correct in Play-In-Editor          │
  │  ← THIS GATE IS MISSING AND MUST BE ADDED                  │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼
  stare-sync reconcile: behavior delta → timeline_integrity_log
```

### Test Diagram
| Codepath | Test Type | Exists? | Gap |
|----------|----------|---------|-----|
| MCP server startup + port detection | Smoke: `python3 stare_mcp_server.py &` → `is_editor_alive()` | NO | Add to Phase 1 checklist |
| RC class search with fully-qualified path | Integration: `search_assets("/Script/STARE.GauntletTierRow")` | NO | Add to Phase 1 checklist |
| Gauntlet tier promotion (BCT 1→2, BIR gate) | PIE: set BCT attr, call RefreshTier, verify ActiveTier | NO | Add to Phase 4 criteria |
| Tier promotion on fresh world (null DataTable) | PIE: no saved state → expect None tier not crash | NO | Add to Phase 4 criteria |
| Double-promote guard (RefreshTier called twice same frame) | PIE: rapid BCT changes → verify single transition event | NO | Add to Phase 4 criteria |
| DataTable hot-reload during PIE | PIE: reimport CSV while playing → verify no stale pointer crash | NO | Add to Phase 4 criteria |
| stare-sync add <name> → character_archives.json | Unit: parse result JSON, verify fields present | NO | Add to Phase 3 checklist |
| Port-file race on Aura restart | Integration: restart Aura mid-session → verify retry succeeds | NO | Document in mcp_server.py |

### NOT in scope (auto-deferred)
- Notion population → PLAN-stare-notion-character-population.md
- Git-LFS / source control strategy → defer to separate plan
- UE5.9 migration path → pin note in TODOS.md, revisit on engine update
- Co-developer onboarding documentation → defer to external collaborator bible
- Multiplayer / replication → explicitly blocked in Phase 1

### What already exists
- `Tools/stare_mcp_server.py` — FastMCP server (15 tools, port-file detection, UE5 Python 3.11)
- `~/.claude.json` — stare-ue5 registered
- `~/.claude/skills/stare-ue5-mcp/SKILL.md` — registration + troubleshooting
- `~/.claude/skills/ue5-stare/SKILL.md` — full harness (API table, build commands, error registry)
- `~/.claude/skills/stare-sync/SKILL.md` — 6-step pipeline
- `Source/STARE/STAREGauntletTypes.h` — FGauntletTierRow, FGauntletEquipState, EGauntletTier
- `Source/STARE/STAREGauntletComponent.h/cpp` — full component implementation
- `Content/Data/DT_GauntletTiers_VhrexKaulvoids.csv` — 5-tier DataTable
- `Tools/setup_gauntlet_component.py` — automates DataTable import + BP wiring

### Amendments from review
The following items are added to the plan based on review findings (auto-decided, P1/P2):

1. **Add PIE validation gate to Phase 4 loop** (CEO critical): Before stare-sync writes
   any mechanic to canon, human confirms behavior in PIE. Added to Phase 4 step 5.

2. **Add port-file race fix to stare_mcp_server.py** (Eng high): Wrap every RC HTTP call
   in retry-with-reread on ConnectionRefused. Max 3 retries, 2s backoff.

3. **Add RC session token** (Eng critical): Write a session token to the port file.
   stare_mcp_server.py echoes it on every RC call. Minimum viable auth on localhost.

4. **Add `Tools/schemas/character_archive_v1.json`** (Eng high): jsonschema contract
   between stare-sync output and MCP DataTable import. Validation runs at loop entry.

5. **Add cold-start section** (DX critical): 5-step path from clean install to first
   `is_editor_alive()` call. Target TTHW: 10 minutes.

6. **Add timeout/hang error row to Failure Modes table** (DX critical): RC calls hang
   when editor frozen (not closed). Timeout: 30s. Retry: 3x with 5s backoff.

## Decision Audit Trail

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|----------------|-----------|-----------|---------|
| 1 | Phase 1 | Use UE5 Python 3.11 not system Python | Mechanical | DRY + explicit | pydantic_core ABI is compiled for exact Python minor version; no workaround | System Python shim |
| 2 | Phase 1 | stdio MCP transport (not HTTP) | Mechanical | Explicit over clever | Claude Code spawns automatically; no port management | HTTP transport |
| 3 | Phase 2 | DataTable for tier config (not hardcoded) | Taste→auto | Completeness | Designer-editable without recompile; follows existing project convention | Hardcoded structs |
| 4 | Phase 3 | File-wins reconciliation in stare-sync | Mechanical | Bias toward action | Git is the truth; seed canon is a starting point only | Seed-wins |
