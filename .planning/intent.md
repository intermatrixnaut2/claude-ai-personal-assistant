# Intent: STARE Tighten-Up Pass — Close All Open C++, DataAsset, and Editor Gaps

**Date:** 2026-09-24
**Author:** intermatrixnaut2
**Branch:** main
**Stage:** 1 — Intent

---

## Problem

The STARE_Viaticum UE5 project has accumulated a backlog of unimplemented or half-implemented systems across three categories: C++ stubs with no corresponding GAS attribute sets or GameplayEffects (VNA/CrownTorque, 6 vessel DataAssets, 2 PlayerState TODOs), editor-only DataAsset instances that can only be created while the editor is running (15 BioEngineering skill nodes + tree, 2 Blueprint steps for Bond Schematic), and creator decisions needed before implementation can proceed (gem_physics values, character stats, character assignments). The master PDF (STARE_Universe_Codex.pdf) documents these systems as canon — they need matching code.

## Context

This is a sequential housekeeping pass, not a new feature. All architectural decisions were made in prior sessions and are documented in the codex PDF, PLAN files, and lore docs. The work is categorized into four waves based on dependency order: pure C++ (no editor needed) → editor/MCP-dependent → creator decisions required → deferred. Each wave is a prerequisite for the next in the editor-dependent track. The UE5 editor gate must be checked before any Wave 2 work.

## Success Criteria

- `USTARENodeAttributeSet` exists in C++ with 7 GAS attributes and compiles clean; 7 corresponding GameplayEffect .h stubs exist
- `STAREPlayerState.h/.cpp` has `bT3VoteInTransit`, `PendingLDDelta`, `IntentLagRemaining` members and the two TODO stubs are resolved
- 6 vessel DataAsset .h files exist (VectorVixin, GorgeQ'ui, Ilastria, VeedleNacht, GhauyKhail, XichuinGrexelFlu) following the GhaisseNhechtDataAsset.h pattern, with placeholder gem_physics values and TODO comments for creator fill-in
- Build.sh exits clean (Result: Succeeded) after all Wave 1 C++ changes — no compile failure on next editor restart
- DA_BioEng_* DataAsset instances (15 nodes + 1 tree) exist in the Content Browser (Wave 2 — requires editor running)
- BP_BondSchematicHologram and WBP_BondSchematic Blueprints are created with the required named widgets (Wave 2 — requires editor running)

## Constraints

- Wave 2 (editor/MCP) blocked until Wave 1 C++ builds clean AND editor is confirmed running via `pgrep -x UnrealEditor`
- Wave 3 items (gem_physics values, character stats) are blocked until creator makes decisions — flag clearly, do not invent values
- No new UCLASS/UENUM introductions that break existing reflection without verified build
- UHT stale cache fix: always clear `Intermediate/Build/Mac/UnrealEditor/Inc/STARE/` before building after adding new USTRUCT/UCLASS headers
- Character names must come from `docs/lore/source/CHARACTER _ VEHICLE.pdf` — do not invent
- Never delete or overwrite existing DataAsset content

## Non-Goals

- DT_PlanarPlanes 17→28 Outer Ring GEM expansion (deferred — Wave 4)
- Cosmo-Helix HTML AstroCity node (deferred — Wave 4)
- Full Blueprint graph logic, AI behavior trees, UI widget gameplay code
- gem_physics value fills for the 6 new vessel DataAssets (creator decision required first)
- KG triple population (separate pass)

## Exploration Hints

- VNA/CrownTorque spec: `docs/lore/STARE-VNA-CrownTorque.md:256-278` — 7 attrs, 7 GEs, 3 PlayerState fields
- gem_physics pattern: `STAREGhaisseNhechtDataAsset.h` — Military_FieldStrength, Military_ThirringHarvestRate, Civilian_FieldStrength, GEM_Tier, GEM_CoherenceThreshold
- Bond Schematic open steps: `project_stare_bond_schematic_todo.md` in memory
- BioEng DataAsset script target: `STAREBioEngineeringSkillNode.h` + `STAREBioEngineeringSkillTree.h`
- PlayerState TODOs at lines ~281 and ~308 in `STAREPlayerState.cpp`
- Build command: `"/Users/Shared/Epic Games/UE_5.8/Engine/Build/BatchFiles/Mac/Build.sh" STARE_ViaticumEditor Mac Development "…/STARE_Viaticum.uproject"`
- UHT cache clear: `rm -rf Intermediate/Build/Mac/UnrealEditor/Inc/STARE/`

---

*Artifact: AI-native SDLC Stage 1 — Intent*
*Skill: /intention*
*Committed artifact chain: intent.md → design-doc → plan → diff+tests → PR → incident-record*
*Next stage: plan execution — Wave 1 C++ first, then Wave 2 with editor gate check*
