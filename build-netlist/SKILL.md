---
name: build-netlist
description: Systematically build SPICE netlists from circuit boards through iterative tracing and verification. Use when reverse-engineering circuits from physical descriptions, building netlists incrementally without schematics, verifying netlist accuracy against board layout, or documenting RC networks and power delivery chains. Provides 6-step workflow with explicit connection documentation, functional naming conventions to prevent junction bloat, QSPICE simulation guidance, common mistakes with fixes, and LLM-specific error patterns from Auto-SPICE and SPICEAssistant research.
allowed-tools: [Read, Write, Edit, Bash, Grep, Glob, WebFetch, WebSearch]
version: 20251109
---

# Building SPICE Netlists from Circuit Descriptions

## What This Skill Does

Build SPICE netlists by tracing connections from physical boards or descriptions. Follow a 6-step process emphasizing explicit documentation, functional naming, and verification.

**Core principle:** Use disciplined naming, explicit connections, and systematic verification to ensure unambiguous, simulation-ready topology.

## When to Use This Skill

- Reverse-engineer circuits from physical descriptions or board images
- Build netlists incrementally without schematics
- Verify netlist accuracy against board layout
- Document multi-stage RC networks, buffer stages, or power delivery chains
- Simulate and validate circuits in QSPICE

## Quick Start: 6-Step Process

For detailed guidance, see [Process Steps](./references/process-steps.md).

**1. Initialize Tracking Table** - Document all connections as you trace (both pads for every component)

**2. Ask Three Questions** - For each component: "What connects to side 1?", "What connects to side 2?", "Same junction or different?"

**3. Validate Junctions** - Ask "What else is here?" before accepting any connection (prevents junction bloat)

**4. Document Explicitly** - Say "R2's left pad connects to R1's right pad" not "R2 connects to R1"

**5. Name Nodes Functionally** - Use GATE_BIAS, Q7_DRAIN, not NODE_A (2-4 components per junction is typical)

**6. Output and Verify** - Read back all components and junctions. Run QSPICE; verify no convergence errors, floating nodes, or extreme voltages

## SPICE Component Syntax Reference

| Component | Syntax | Example |
|-----------|--------|---------|
| **R** | R_NAME NODE1 NODE2 VALUE | R_Gate C1P Q7_Gate 10k |
| **C** | C_NAME NODE1 NODE2 VALUE | C_Bypass Q7_Gate GND 100n |
| **D** | D_NAME ANODE CATHODE MODEL | D_Freewheel Q7_Drain Q7_Source D_1N4148 |
| **Q** | Q_NAME C B E MODEL | Q_Driver Q_Drain Gate_Bias GND Q_2N3906 |
| **M** | M_NAME D G S B MODEL | M_Q7 Q7_Drain Q7_Gate Q7_Source Q7_Source NMOS_2N03L05 |
| **V** | V_NAME NODE+ NODE- VALUE | V_Alt C1P GND DC 12 |
| **I** | I_NAME NODE+ NODE- VALUE | I_Load Out GND DC 100m |
| **B** | BX_NAME NODE+ NODE- X=EXPR | BV_FB FB GND V=V(VOUT)/10 |

### ⚠️ CRITICAL UNIT WARNING

**1M = 1 milliohm, NOT 1 megaohm.** This is the #1 unit mistake in SPICE netlists.

- ❌ `R_Feedback VCC FB 1M` → 1 milliohm (nearly a short)
- ✅ `R_Feedback VCC FB 1Meg` → 1 megaohm (correct)

**All engineering multipliers:** T (1e12), G (1e9), **Meg (1e6)**, K (1e3), M (1e-3), u/μ (1e-6), n (1e-9), p (1e-12), f (1e-15)

**Why:** The "M" multiplier predates modern convention. Always use "Meg" or spell out "1000000" for clarity.

---

## ⚠️ CRITICAL: Error Handling Protocol for Reverse Engineering

**Two fundamental adjustments to LLM behavior (v20251109 - CRITICAL UPDATE):**

### Adjustment 1: Don't Assume Previous Work Is Correct

Errors are **equally likely** in past work as in current work. When you encounter previous documentation or netlist entries:

- **Validate against current observations** - Ask: "The netlist shows [X]. What do you observe on the physical board?"
- **Treat all entries as potentially wrong** - A connection documented in a prior session may have been traced incorrectly
- **Build confidence incrementally** - Only accept connections as verified once physically confirmed with the user

### Adjustment 2: Don't Fix "Errors"—Consult the User

**DO NOT auto-correct anomalies.** Preserve them and ask the user what to do.

| ❌ Wrong | ✅ Correct |
|---------|----------|
| See R18=712MΩ looks unreasonable → silently "fix" it | See R18=712MΩ → "This is unusually high. Should we investigate or accept as-is?" |
| Find unusual Q12 config → change it to "proper" circuit theory | Find unusual Q12 config → "This is uncommon. Does it tell us something about the design?" |
| Encounter D9 as wire jumper (not diode) → correct it | Encounter D9 as wire jumper → "The footprint shows a diode, but there's a wire jumper. Why?" |

**Why:** Anomalies are information. They reveal design intent, constraints, cost optimization, or field mods. By fixing errors, you hide the clues that lead to understanding.

**When anomalies appear:**
1. Acknowledge explicitly: "R18 is 712MΩ, which is unusually high for a resistor divider"
2. Describe context: "Typically values are in the kΩ range; this is 1000× larger"
3. Consult: "Is this correct? Should we investigate, defer, or treat as data?"
4. Document: Add netlist comment explaining the anomaly for future sessions

**Waiting for more information is always valid.** It's acceptable to mark something "needs investigation" and move forward.

---

**See:** [Error Handling Protocol](./references/error-handling-protocol.md) for complete guidance on discrepancies, validation, and anomaly handling.

---

## ⚠️ CRITICAL: Authority Hierarchy - Physical Board Is Truth

**The physical board is the authoritative source of truth. The netlist should be the authoritative documentation of that board. All other documents are subordinate.**

### Authority Hierarchy

| Source | Authority | Responsibility |
|--------|-----------|-----------------|
| **Physical board** | ✅ TOP | What actually exists |
| **SPICE netlist** | ✅ AUTHORITATIVE DOCUMENTATION | Should accurately represent the physical board |
| **Documentation files** | ⚠️ SUBORDINATE | Must match netlist and board |
| **Comments in netlist** | ⚠️ SUBORDINATE | Navigation aid only, never override elements |
| **External schematics** | ⚠️ SUBORDINATE | Design intent, but board/netlist may differ |

### When Conflicts Arise

**Physical board vs. Netlist conflict:**
- ❌ DO NOT change the netlist to "match" documentation
- ✅ DO flag to the user: "Physical board shows [X] but netlist shows [Y]. Which is correct? Netlist may need correction."
- The netlist failed to capture the board accurately and requires user verification

**Netlist vs. Documentation conflict:**
- ✅ Update documentation to match netlist
- ✅ The netlist is the authoritative documentation of the board

---

## ⚠️ CRITICAL: Netlist Comment Policy

**Comments in SPICE netlists become outdated and incorrect as the circuit evolves.** Do NOT treat comments as finished work—they are the FIRST things to become wrong during revision.

### Default Comment Policy

**By default, minimize or eliminate inline comments** and rely on functional node names for self-documentation. However:

- **Comments are kept at user's option** - If the user wants comments in the netlist, honor that choice
- **Comments are never authoritative** - The netlist elements (resistor values, node connections) are always the source of truth
- **Conflicts must be reported** - If any comment contradicts the netlist elements, **explicitly flag this to the user** before proceeding

### When to Flag Comment/Code Conflicts

**Always point out conflicts explicitly:**

- ❌ A comment says "R1 connects to Q13_BASE" but netlist shows `R1 P1_PIN10 Q13_EMIT`
- ❌ A comment says "D12 cathode on Q13_BASE" but netlist shows `D12 Q10_Q13_COLL Q10_EMIT_NODE`
- ❌ A comment describes "unpopulated R2" but netlist has `R2 NODE1 NODE2 VALUE`

**Always ask the user:** "Comment says [X] but netlist shows [Y]. Which is correct? Should we update the comment or the netlist?"

### Recommended Comment Guidelines (If Keeping Comments)

1. **Section headers only** - Use `* ============================================================================` to organize signal flow
2. **NO inline component comments** - Avoid `R1 P1_PIN10 Q13_BASE 10 ; input to Q13 base`
3. **NO topological documentation** - Don't use comments to explain connections
4. **NO historical notes** - Don't use `* FIXED 2025-11-10:` or `* TODO:` in netlists

### Why Comments Become Wrong

During revision cycles:
- The circuit changes (node names, connections, values)
- Comments are NOT updated
- Future work trusts outdated comments
- Errors compound invisibly

**Solution:** Use **functional node names** (Q10_Q13_COLL, Q10_EMIT_NODE) to make topology self-documenting. If you need a comment to explain what a node does, rename the node instead.

### Template (Minimal Comments)

```spice
* ==============================================================================
* POWER STAGE
* ==============================================================================
V_ALT C1P 0 DC 12V
C1 C1P 0 10000uF IC=12V

* ==============================================================================
* GATE DRIVE CHAIN
* ==============================================================================
Q_Q13 Q10_Q13_COLL Q13_BASE Q13_EMIT Q_TIP105_PNP
R1 P1_PIN10 Q13_BASE 10
```

**Functional node names eliminate the need for inline documentation.** If the user requests comments, keep only section organization headers—never let comments become alternative documentation that can diverge from the actual netlist.

---

## Skill Scope & Limitations

**What This Skill Does Well:**
- Reverse-engineer circuits from **physical descriptions or board images**
- Build netlists **incrementally without schematics**
- Document **gate drivers, feedback networks, and power delivery chains**
- Validate circuits in **QSPICE with behavioral sources and optimization**

**What This Skill Cannot Do:**
- **Analyze PCB images** — requires user-provided circuit descriptions
- **Verify against actual hardware** — outputs simulation netlists only
- **Extract schematics from photos** — requires described circuits, not images
- **Handle multi-layer boards without layer information** — single-layer or simple multi-layer only
- **Identify unlabeled ICs** — requires component part numbers or functional descriptions

**Best Results When:**
✅ Components are labeled or identified
✅ Board connections are visible from one side or clearly described
✅ All signal paths are documented
✅ Building simulation models for validation

**May Struggle With:**
⚠️ Ambiguous component layouts with overlapping connections
⚠️ Hidden layers or buried vias without explicit documentation
⚠️ Unknown components without part numbers or datasheets
⚠️ Multi-chip designs without sub-circuit breakdowns

---

## File Organization for Complex Projects

For boards with multiple functional blocks, organize netlists hierarchically:

**Recommended folder structure:**
```
project/
├── main_circuit.cir              # Top-level netlist
├── models/
│   ├── transistors.lib           # MOSFET/BJT models
│   ├── regulators.lib            # Regulator IC models (LM1086, etc.)
│   └── subcircuits/
│       ├── gate_driver.cir       # Gate driver stage
│       ├── feedback.cir          # Feedback network
│       └── power_stage.cir       # Power switching stage
├── analysis/
│   ├── transient.cir             # Transient (time-domain) analysis
│   ├── ac_response.cir           # AC frequency response
│   └── tolerance.cir             # Monte Carlo/worst-case analysis
└── documentation/
    ├── connections.md            # Connection table
    └── junctions.md              # Junction documentation
```

**Using .include to structure netlists:**

```spice
* Main Circuit File
.title Life Fitness Controller Board

* Load external models
.include models/transistors.lib
.include models/regulators.lib

* Define subcircuits
.include models/subcircuits/gate_driver.cir
.include models/subcircuits/feedback.cir

* Main circuit instantiates subcircuits
X_Gate_Driver PWM_IN Q7_GATE GND gate_driver_block
X_Feedback OUT FB GND feedback_network

* ... rest of main circuit
```

**Benefits of hierarchical organization:**
- Swap component models easily
- Reuse subcircuits across designs
- Clarify circuit intent
- Simulate individual blocks independently
- Scale to multi-block designs

---

## The 10 Essential Rules (+ NC Naming)

These separate good netlists from broken ones. See [Pithy Rules](./references/pithy-rules.md) for full explanations.

1. **Explicit Over Implicit** - Say which side and which junction, never assume
2. **Document All Terminals** - Every terminal of every component must be documented (not just "two sides")
   - Two-terminal: R, C, L, D (2 pads each)
   - Three-terminal: BJT (3 pins)
   - Four-terminal: MOSFET (D, G, S, B)
   - N-terminal: ICs (N pins)
   - Each terminal must be: (1) connected to node, (2) deliberately NC with unique name, or (3) tied to power/ground
3. **Vague Names = Wrong Netlist** - Use GATE_BIAS not NODE_A (2-4 components per junction)
4. **Ask "What Else is Here?"** - Prevent junction bloat by validating before confirming
5. **Physical Reference System** - Pick left/right or top/bottom, use consistently
6. **Direction Matters** - Series vs parallel vs bias divider—getting this wrong breaks the circuit
7. **DC Path to Ground Required** - Every node must have DC path to GND (SPICE requirement)
8. **Hierarchical Organization Scales** - Use sections and `.include` for complex boards
9. **Names Describe Function** - GATE_BIAS (what it does) not R1_R2_JUNC (location)
10. **Verify Before Finalizing** - Read back all components, verify topology makes sense
11. **Unique NC Names** - Deliberately unconnected pads get unique names: `NC_[COMPONENT]_[PAD]` (not generic `NC` or `NC1`)

---

## Key Reference Files

Load these files as needed based on your task:

- **[Process Steps](./references/process-steps.md)** - Detailed 6-step workflow with questions to ask and checklists
- **[Pithy Rules](./references/pithy-rules.md)** - The 10 essential rules with explanations and why they matter
- **[Naming Conventions](./references/naming-conventions.md)** - Component and node naming strategy, preventing junction bloat
- **[QSPICE Guide](./references/qspice-guide.md)** - Running QSPICE on WSL, behavioral sources, simulation validation
- **[Common Mistakes](./references/common-mistakes.md)** - 12 common netlist errors with fixes and prevention
- **[Example Netlists](./assets/example-netlists.md)** - Working examples from real circuit tracing

---

## QSPICE Behavioral Sources (Unique Feature)

QSPICE allows arbitrary mathematical expressions in netlists:

```spice
BV_FB FB GND V=V(VOUT)/10              * Voltage division
BI_SENSE Sense GND I=I(R_Shunt)        * Access resistor current
BR_Dynamic N1 N2 R=1k + V(Ctrl)*100    * Voltage-dependent resistor
```

See [QSPICE Guide](./references/qspice-guide.md) for complete documentation and examples.

---

## Running QSPICE on WSL/Debian

```bash
'/mnt/c/Program Files/QSPICE/QSPICE64.exe' your_circuit.cir
```

See [QSPICE Guide](./references/qspice-guide.md) for error troubleshooting and best practices.

---

## Common Mistakes to Avoid

**Before finalizing, check for these:**

1. Half-traced components (missing one pad)
2. Ambiguous side references ("R1 connects to R2" without specifying sides)
3. Generic node names creating junction bloat (NODE_A with 8+ components)
4. Missing DC paths to ground (floating nodes)
5. Series/parallel confusion (not tracing both pads to verify)
6. Location-based names instead of function-based (R1_R2_JUNC not GATE_BIAS)
7. Wrong MOSFET pin order (should be D-G-S-B)
8. Inconsistent case in node names (gate_bias vs GATE_BIAS)

See [Common Mistakes](./references/common-mistakes.md) for detailed fixes and prevention strategies.

---

## Standards & References

No IEEE standard exists for SPICE (implementations: SPICE2, SPICE3, LTspice, ngspice, QSPICE). All modern variants are SPICE3-compatible.

**External reference sources:**
- SPICE Fundamentals: ecircuitcenter.com, allaboutcircuits.com
- Academic Guides: MIT, Stanford SPICE tutorials
- QSPICE Documentation: Qorvo official docs and tech forum (forum.qorvo.com)
- EDA Tools: Altium, SIMetrix, ngspice manuals

---

## Academic Research & Validation

This skill incorporates findings from recent peer-reviewed research on LLM-assisted circuit design:

**Core Research:**
- **Auto-SPICE** (arXiv 2411.14299v1) - 46% accuracy improvement with structured component detection and prompt engineering
- **SPICEAssistant** (arXiv 2507.10639v1) - 38% improvement over GPT-4o using simulation feedback loops
- **SPICEPilot** (arXiv 2410.20553v1) - Complexity-tiered benchmarking for netlist generation

**Industry Standards Referenced:**
- **Cadence Design Systems** - Hierarchical naming conventions, pin naming limits
- **Intel/Altera** - Node naming standards, functional block organization
- **IPC Standards** - Design rule checks, verification workflows for hardware reverse engineering
- **Qorvo Technical Forum** - QSPICE optimization techniques, behavioral source performance

**Why These Practices Work:**
- Explicit naming reduces ambiguity by 46% (Auto-SPICE validation)
- Iterative simulation feedback improves accuracy by 38% (SPICEAssistant)
- Floating nodes are the #1 LLM-generated error (prevents 30% of netlist failures)
- 5% tolerance metrics more reliable than exact parameter matching for validation

---

## LLM Research Insights

Recent academic research on LLM-generated netlists validates the skill's approach:

**Key Papers:**
- **Auto-SPICE** (Masala-CHAI framework): 46% improvement with component detection + prompt engineering + verification
- **SPICEAssistant**: 38% better than GPT-4o on switched-mode power supplies using simulation feedback
- **SPICEPilot**: PySpice + LLM integration with benchmarks across complexity tiers

**Common LLM Netlist Errors:**
- MOSFET drain/source terminal swapping
- Incorrect connection assumptions at non-junction intersections
- Differential pair input/output mishandling
- Missing components in dense layouts

**Why It Works:**
- **38% accuracy improvement** with simulation feedback loop (build → test → iterate)
- Floating nodes are the #1 error (why Rule 7 is non-negotiable)
- Explicit naming reduces ambiguity dramatically (functional names prevent interpretation errors)
- Structured verification beats intuition
- Graph-based validation (Graph Edit Distance) catches topology errors

**Implication:** Build iteratively, test with QSPICE, refine based on simulation results. Research shows 5% tolerance metrics for parameter validation work better than exact matches.

---

## Version History

**v20251109** - Critical behavioral adjustments: (1) Don't assume previous work is correct—validate against current observations, (2) Don't fix "errors" without consulting user—preserve anomalies as information. See `/references/error-handling-protocol.md`.

**v20251107** - Core 6-step process, 10 essential rules, common mistakes reference, QSPICE guide, research insights.

---

## Academic References & Research Validation

This skill is grounded in peer-reviewed research on LLM-assisted circuit design and industry best practices.

### Primary Academic Papers

**1. Auto-SPICE: Automatic SPICE Netlist Generation Using Open-Source Tools**
- **Citation:** arXiv:2411.14299v1 (November 2024)
- **Key Finding:** 46% accuracy improvement through structured component detection + prompt engineering + verification
- **Validates:** 3-step workflow, graph-based topology validation, explicit naming reduces ambiguity
- **Application:** Component detection framework, iterative verification process

**2. SPICEAssistant: Design-Centric Approach to Using Large Language Models for Circuit Simulations**
- **Citation:** arXiv:2507.10639v1 (July 2025)
- **Key Finding:** 38% improvement over GPT-4o baseline using simulation feedback loops on switched-mode power supplies
- **Validates:** Iterative build → test → refine workflow, 5% tolerance metrics for validation
- **Application:** Simulation-based verification, closed-loop netlist improvement

**3. SPICEPilot: Automatic SPICE Netlist Generation for Placement-Driven Layout Optimization**
- **Citation:** arXiv:2410.20553v1 (October 2024)
- **Key Finding:** Complexity-tiered benchmarking, Pass@k metrics for netlist generation
- **Validates:** Component parameter accuracy, gate width/length ratios for MOSFETs
- **Application:** Netlist accuracy metrics, complexity classification

### Industry Standards & Best Practices

**Cadence Design Systems**
- HSPICE documentation and design rules
- Hierarchical naming conventions (BLOCK.SUBBLOCK.SIGNAL)
- Pin naming constraints (max 16 characters)
- "Name everything" design principle

**Intel/Altera Quartus Design**
- Node naming standards for large-scale designs
- Hierarchical organization techniques
- Multi-layer netlist management

**IPC Standards (Institute for Interconnecting and Packaging Electronic Circuits)**
- Design Rule Checks (DRC) for netlist verification
- Reverse engineering best practices
- Quality Management Systems for hardware documentation

**Qorvo Technical Forum & Documentation**
- QSPICE-specific optimization techniques
- Behavioral source performance guidelines
- Integration method selection (trapezoid vs gear)
- .bode analysis for switching circuits

### Key Validation Points

| Principle | Research Source | Evidence |
|-----------|-----------------|----------|
| **Functional naming reduces errors** | Auto-SPICE | 46% improvement with explicit naming |
| **Floating nodes are #1 LLM error** | SPICEAssistant | 30% of failures due to missing DC paths |
| **Iterative feedback improves accuracy** | SPICEAssistant | 38% improvement with simulation loop |
| **5% tolerance metrics reliable** | SPICEAssistant | Better than exact parameter matching |
| **Graph-based validation catches topology errors** | Auto-SPICE | Graph Edit Distance validation |
| **2-4 components per junction typical** | Industry practice | Prevents junction bloat errors |
| **DC path to ground mandatory** | SPICE fundamentals | Non-negotiable for convergence |
| **Hierarchical organization scales** | Cadence, Intel | Best practice for complex designs |

### How to Use These References

**For academic validation:**
- Cite Auto-SPICE (2411.14299v1) when discussing naming conventions and component detection
- Cite SPICEAssistant (2507.10639v1) when emphasizing simulation feedback and tolerance analysis
- Cite SPICEPilot (2410.20553v1) when discussing netlist complexity and parameterization

**For industry standards:**
- Reference Cadence HSPICE docs for hierarchical naming and 16-character pin limit
- Reference IPC standards for DRC and reverse engineering workflows
- Reference Qorvo forum for QSPICE-specific techniques

**For methodology validation:**
- The 6-step process aligns with Auto-SPICE's 3-step framework (detection → engineering → verification)
- The 10 Essential Rules encode lessons from 25+ years of SPICE netlist best practices
- Graph-based validation technique from Auto-SPICE is optional but recommended for complex designs

