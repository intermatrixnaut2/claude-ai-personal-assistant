<!-- /autoplan restore point: "/Users/intermatrixnaut/.gstack/projects/intermatrixnaut/main-autoplan-restore-20260923-195321.md" -->
# PLAN: STARE Bio-Engineering Skill Tree
## Active DNA Expression Alteration + Morphogenetic Field Canon + UE5 DataAsset Stubs

**Status:** DRAFT  
**Date:** 2026-09-23  
**Intent:** `.planning/intent.md`  
**Source video:** `/Users/intermatrixnaut/Downloads/tiktokio.world_Biology_is_starting_to_look_a_lot_less_like_blind_machinery_.mp4`  
**Scientific basis:** Michael Levin — *Trends in Genetics* (CellPress), "What does evolution make? Learning in living lineages and machines"  
**Extends:** `docs/lore/STARE-TrooperBiofieldProtocol.md`  
**UE5 anchors:** `Source/STARE/STARELymnoticComponent.h`, `Source/STARE/STARESpermatogenicComponent.h`

---

## Problem

The STARE Trooper Biofield Protocol defines the *physics* of trooper biology — BIR, VhexFluid, Biological STAR Oscillators, voltage doctrine. It has no *programmability* layer: no mechanic for troopers who consciously alter their own DNA expression and body form. The video establishes that biology is not blind machinery — it is goal-directed, information-processing, and navigates toward target configurations. This is exactly the missing STARE canon pillar: how troopers learn to consciously author their own biological program.

## Scientific Extraction (video-grounded)

### Core Levin Claims Witnessed in Video

| Frame Subtitle | Scientific Concept | Status |
|---|---|---|
| "BIOLOGY IS GETTING WEIRD... AND MATERIALISTS ARE SQUIRMING" | Biology as cognitive agency, not mechanism | Canonized |
| "THE ANATOMY, THE TRAITS, AND PHENOTYPE" | Genotype != phenotype; information layer between them | Canonized |
| "AND PROCESS INFORMATION" | Cells as computational nodes | Canonized |
| "CALLED MORPHOGENESIS" | Form-building as active navigational process | Canonized |
| "REACH PARTICULAR TARGET MORPHOLOGIES" | Goal-directedness in cell collective behavior | Canonized |
| "AND USE DIFFERENT MEANS" | Equifinality — multiple paths, same target form | Canonized |
| "STRUCTURE WITH LESS CELLS" | Robust regeneration: correct form even with fewer cells | Canonized |
| "ABOUT THE BODY PLAN" | Body plan as reprogrammable software, not fixed hardware | Canonized |
| "REGENERATE WITH TWO HEADS" | Levin planaria experiment: block gap junctions = wrong body plan | Canonized |

### The Core Insight for STARE

The DNA sequence is **hardware**. The bioelectric state of the cell collective is **software** — it encodes the target body form (Form Anchor) and every cell continuously reads it to know what to build. Changing the software changes what the cells build, without touching the DNA. This is the mechanism STARE bio-engineers exploit.

---

## STARE Terminology Canonization

| Real Concept | STARE Canon Term | Definition |
|---|---|---|
| Morphogenetic field | **VhexForm Field** | Bioelectric information layer encoding the body's target configuration. Distinct from BIR (charge amplitude) — this is the *pattern* that BIR maintains. |
| Target morphology | **Form Anchor** | The body configuration that cell collectives navigate toward and continuously restore. Strong Form Anchor = robust regeneration. Corrupted Form Anchor = aberrant growth. |
| Equifinality | **Path Divergence** | Multiple biological strategies that converge on the same Form Anchor. A trooper with 30% cell loss still reaches correct form if their Form Anchor is intact. |
| Physiological software | **BioScript** | The information layer between DNA (hardware) and body form (output). What STARE bio-engineers actually rewrite. DNA stays identical; BioScript changes. |
| Gap junction signaling | **VhexLine** | Cell-to-cell bioelectric communication channel carrying Form Anchor coordinates. VhexLine Inversion = cells lose their reference — CAINAX weapon. |
| Morphogenesis | **Form-Casting** | The active, continuous process by which cell collectives navigate toward the Form Anchor. Never stops in living troopers. Conscious Form-Casting is Tier III. |
| Cognitive agency of cells | **Cell Sentience** | Each cell is a sub-agent: receives Form Anchor signals, makes local decisions, navigates. BIR is the aggregate coherence of Cell Sentience. |
| Regeneration | **Form-Recall** | The process by which a damaged trooper's cells return to Form Anchor. Accelerated Form-Recall is a trainable skill. |
| Epigenetic methylation | **VhexMark** | Chemical flag written onto the helix that silences or activates gene expression blocks. Does not alter sequence. Persists across cell division. |
| Histone modification | **Oscillator Wrap** | Protein spools that compress (silence) or expose (activate) helix segments. Modifying Oscillator Wrap unlocks dormant Lattice Memory nodes. |
| Signal transduction cascade | **Cascade Glyph** | A chain-trigger event: one external signal propagates through a cell into coordinated gene expression shift. Cascade Glyph Trigger is Tier IV. |
| Gene regulatory network | **Lattice Circuit** | Web of interdependent gene switches below conscious BIR. Bio-engineering at Tier III reads and selectively rewires Lattice Circuits. |
| Optogenetic control | **Photon-Command** | Light-wavelength-triggered gene activation. Connects to LWM Triad Light pillar. |
| mRNA splicing variants | **Splice Fork** | Same gene, different protein output based on context. Splice Fork Selection (APEX) is conscious choice of variant. |
| Evolutionary learning | **Lattice Memory** | Accumulated bio-engineering knowledge encoded through evolutionary time. Dormant capability nodes unlocked by Oscillator Wrap Release. |
| Gap junction block / two heads | **VhexLine Inversion** | CAINAX attack: corrupt VhexLine channel — cells lose Form Anchor reference — build against correct body plan. Aberrant growth, organ duplication, structural collapse. |

---

## Implementation plan

### Phase 1: Canon Lore Doc — `docs/lore/STARE-BioEngineeringSkillTree.md`

**Deliverable:** Full canon document with 4 branches, 13 skill nodes, CAINAX offensive use, trooper-facing doctrine.

**Skill Tree Structure:**

```
BRANCH A: VhexForm Field (Bioelectric Mastery)
  I    VhexLine Reading        — sense own bioelectric field state; baseline Form Anchor assessment
  II   VhexLine Amplification  — strengthen cell-to-cell signal; faster Form-Recall after injury
  III  VhexLine Projection     — extend field beyond skin; affect adjacent biological matter
  APEX Form Anchor Override    — temporarily write a new Form Anchor target

BRANCH B: Form-Casting (Morphogenetic Navigation)
  I    Form Anchor Lock        — resist VhexLine Inversion; reinforce Form Anchor stability
  II   Accelerated Form-Recall — compress regeneration timeline; rebuild with fewer cells
  III  Form Deviation          — shift Form Anchor to alternate configuration; controlled self-modification

BRANCH C: BioScript Alteration (Epigenetic Programming)
  I    BioScript Read          — identify active gene expression states; self-diagnostic
  II   VhexMark Write          — place/erase chemical flags on helix segments
  III  Oscillator Wrap Release — decompress histone-wrapped segments; access Lattice Memory
  IV   Cascade Glyph Trigger   — activate signal transduction chains from environmental cue
  APEX Splice Fork Selection   — choose mRNA splice variant consciously; same gene, different protein

BRANCH D: Cell Sentience (Cognitive Agency Expansion)
  I    Cell Sentience Attunement  — sync conscious awareness with cellular intelligence; BIR +5/tier
  II   Lattice Memory Access      — read evolutionary-encoded patterns; unlock dormant Form-Cast configs
  III  Photon-Command             — use light wavelength as BioScript trigger; LWM Triad integration
```

**Steps:**
- [x] 1.1 Write `docs/lore/STARE-BioEngineeringSkillTree.md` (full doc, all 4 branches, all 15 nodes with lore text)
- [x] 1.2 Add VhexLine Inversion as CAINAX weapon in offensive doctrine section
- [x] 1.3 Add cross-reference line to `docs/lore/STARE-TrooperBiofieldProtocol.md`
- [x] 1.4 Add lore doctrine headers to `STARELymnoticComponent.h` and `STARESpermatogenicComponent.h`

---

### Phase 2: STARE Sync — Index in Knowledge Base

**Deliverable:** Video concepts indexed in docs/lore/memory_index.md, KG triples written, source citation saved.

**Steps:**
- [x] 2.1 Add entry to `docs/lore/memory_index.md` for STARE-BioEngineeringSkillTree
- [ ] 2.2 KG triple: VhexForm Field extends BIR system
- [ ] 2.3 KG triple: BioScript Alteration requires BioScript Read (prerequisite chain)
- [ ] 2.4 KG triple: VhexLine Inversion is CAINAX attack targeting Form Anchor
- [ ] 2.5 KG triple: Photon-Command integrates LWM Triad Light pillar
- [x] 2.6 Save video source citation as `docs/lore/sources/LEVIN-2026-morphogenetic-learning.md`

---

### Phase 3: UE5 DataAsset Stubs

**Gate:** `pgrep -x UnrealEditor` must be UP before any UE5 work. Editor confirmed UP at session start.

**Deliverable:** C++ DataAsset class stubs. No Blueprint logic. No GAS abilities. Lore-to-code bridge only.

**3.1 — `USTAREBioEngineeringSkillNode` (DataAsset)**

File: `Source/STARE/BioEngineering/STAREBioEngineeringSkillNode.h`

Properties:
- `FName NodeID` — e.g. "VhexLineReading", "SpliceForkSelection"
- `FText DisplayName`
- `FText LoreDescription`
- `ESTAREBioEngineeringBranch Branch` — VhexFormField / FormCasting / BioScriptAlt / CellSentience
- `ESTAREBioEngineeringTier Tier` — Tier_I / Tier_II / Tier_III / Tier_IV / Apex
- `TArray<FName> Prerequisites` — NodeIDs that must be unlocked first
- `float BIRBonusPerLevel` — BIR modifier when node is active
- `float FormRecallAcceleration` — multiplier on regeneration speed
- `bool bUnlocksLatticeMemory` — true only for Oscillator Wrap Release

**3.2 — `USTAREBioEngineeringSkillTree` (DataAsset container)**

File: `Source/STARE/BioEngineering/STAREBioEngineeringSkillTree.h`

Properties:
- `TArray<TObjectPtr<USTAREBioEngineeringSkillNode>> AllNodes`
- `FText TreeLoreCaption` — "The BioScript is not fixed. It was never fixed."

**3.3 — Build.cs**
- `BioEngineering/` subfolder under `Source/STARE/`
- `GameplayAbilities` and `GameplayTasks` already in PublicDependencyModuleNames from Notaric system

**Steps:**
- [x] 3.1 Create `Source/STARE/BioEngineering/` directory
- [x] 3.2 Write `STAREBioEngineeringSkillNode.h` and `.cpp` (stub — reflection only, no logic)
- [x] 3.3 Write `STAREBioEngineeringSkillTree.h` and `.cpp`
- [x] 3.4 Verify STARE.Build.cs includes BioEngineering in source tree
- [x] 3.5 Prompt user to save + restart editor before any Python/Blueprint ops on these types

---

### Phase 4: Cross-Link Notaric System

Notaric and Bio-Engineering are the two parallel "knowledge unlocks capability" pillars. They must cross-reference.

- [x] 4.1 Add one paragraph to `PLAN-stare-notaric-impression-system.md` noting the Bio-Engineering parallel
- [ ] 4.2 Confirm node tier vocabulary matches (I / II / III / APEX) for UE5 consistency

---

## Non-Goals

- No Blueprint logic, no GAS ability trees, no UI widgets this session
- No refactor of STARELymnoticComponent or STARESpermatogenicComponent (additive only)
- No SUNO song or PDF deliverable this session
- No Notaric system changes — distinct canon pillar
- No NFT or blockchain mechanics

---

## What Already Exists

- `docs/lore/STARE-TrooperBiofieldProtocol.md` — Section II: Biological Architecture is the direct parent doc
- `Source/STARE/STARELymnoticComponent.h` — cellular component stub, lore-orphaned, needs doctrine header
- `Source/STARE/STARESpermatogenicComponent.h` — cellular component stub, lore-orphaned, needs doctrine header
- `Source/STARE/NotaricSystem/` — parallel DataAsset pattern; BioEngineering mirrors this structure
- `PLAN-stare-notaric-impression-system.md` — tier vocabulary and UE5 DataAsset pattern to clone

---

## Risk Flags

- **UE5 new reflection types require editor restart** — any session adding UCLASS/UENUM must prompt restart before Python or Blueprint ops
- **STARELymnoticComponent and STARESpermatogenicComponent are built but lore-undefined** — doctrine headers are required before collaborators can make sense of them
- **VhexForm Field is a new top-level system** — it extends BIR but is not BIR; docs must distinguish clearly

---

## Review record

<!-- autoplan-accepted:ceo -->
- VhexForm Field lore doc MUST include a callout box distinguishing it from BIR: "VhexForm Field = pattern (what cells are building toward). BIR = amplitude (how much charge the field carries). They are orthogonal axes. A trooper can have full BIR and a corrupted VhexForm Field simultaneously."
<!-- /autoplan-accepted:ceo -->

<!-- autoplan-accepted:eng -->
- USTAREBioEngineeringSkillNode and USTAREBioEngineeringSkillTree MUST use UPrimaryDataAsset (not UDataAsset) to match Notaric pattern and enable AssetManager registration.
- NodeStatGrants MUST use TMap<FGameplayTag, float> (not individual float fields) to match USTARENotaricPlateDataAsset::ChannelStatBoosts pattern.
- Add TArray<int32> ValidCycleDays property linking Bio-Engineering node timing to STARELymnoticComponent::GetCycleDay().
- Asset naming convention: DA_BioEng_<Branch>_<Tier> (mirrors DA_Plate_<CHANNEL>_<Grade>).
- All UE5 file paths use Unreal Projects/STARE_Viaticum/Source/STARE/BioEngineering/ (not Source/STARE/BioEngineering/).
<!-- /autoplan-accepted:eng -->

## GSTACK REVIEW REPORT

### Review Coverage

| Phase | Native | Outside | Status |
|---|---|---|---|
| CEO | Claude (in-host) | Codex (ready, not dispatched — plan was draft at start) | Complete |
| Design | — | — | Skipped — no UI scope |
| DX | — | — | Skipped — false positive override |
| Eng | Claude (in-host) | Codex (ready, not dispatched) | Complete |

### Decision Audit Trail

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|---|---|---|---|---|---|
| 1 | CEO | VhexForm Field is distinct from BIR | Mechanical | Completeness | Pattern vs amplitude are orthogonal axes — VhexLine Inversion is incoherent without the distinction | Merging them |
| 2 | CEO | 15 nodes at once (not staged) | Mechanical | Completeness | All 15 are video-grounded; partial tree requires second canon pass | Phased node release |
| 3 | Eng | UPrimaryDataAsset over UDataAsset | Mechanical | DRY | Notaric uses UPrimaryDataAsset; AssetManager requires it | UDataAsset |
| 4 | Eng | TMap<FGameplayTag,float> for stat grants | Mechanical | DRY | Matches existing Notaric pattern exactly | Individual float properties |
| 5 | Eng | Add ValidCycleDays property | Mechanical | Completeness | STARELymnoticComponent::GetCycleDay() already exists; connection is natural | Omitting timing link |
| 6 | Eng | UE5 path correction to Unreal Projects/... | Mechanical | Explicit over clever | Plan had wrong base path | Source/STARE/ directly |
