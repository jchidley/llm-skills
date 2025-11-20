# 10 Essential Rules for Netlist Building

These 10 rules separate good netlists from problematic ones. When in doubt during reverse-engineering, fall back on these.

## Rule 1: Explicit Over Implicit

Never assume topology. "R1 connects to R2" is useless. Say which side and which junction.

- ❌ "R1 and R2 are connected" → Ambiguous (series? parallel? which side?)
- ✅ "R1's right pad connects to R2's left pad" → Crystal clear

**Why it matters:** Circuit topology depends entirely on connection direction. Ambiguity = wrong netlist.

## Rule 2: Document All Terminals of Every Component

Don't leave component pads undocumented.

**The underlying problem:** Incomplete tracing causes floating nodes in simulation, which breaks convergence and produces meaningless results.

- ❌ Document: "R5 connects to +5V" (what about the other side?)
- ✅ Document: "R5 left pad → +5V_RAIL, R5 right pad → GATE_BIAS"

**Applies to:**
- **Two-terminal components** (R, C, L, D): 2 terminals each
- **Three-terminal components** (BJT): 3 terminals each
- **Four-terminal components** (MOSFET): 4 terminals each (D, G, S, B)
- **N-terminal components** (IC): N terminals each
- **Single-connection components** (test point, jumper): 1 terminal documented

**For each terminal, specify:**
1. Connected to which node (or NC if no-connect)
2. If NC: use unique name like `NC_R5_PAD2` (deliberate no-connect)
3. Never leave pads undocumented or ambiguous

**Why it matters:** Missing documentation = floating node = simulation failure. Every terminal must either be: (1) connected to a named signal, (2) deliberately NC with unique name, or (3) tied to power/ground explicitly.

## Rule 3: Vague Names = Wrong Netlist

NODE_A creates 10-component junctions. GATE_BIAS keeps it focused (2-4 components typical).

- ❌ "NODE_A" (where did this come from? what's connected?)
- ✅ "GATE_BIAS" or "Q1_DRAIN" (self-documenting and scoped)
- Guideline: 2-4 components per junction is typical; 8+ is a red flag

**Why it matters:** Vague names hide topology errors. When you see NODE_A with 8 things, something is wrong with your understanding.

## Rule 4: Ask "What Else is Here?"

Before accepting "connects to NODE_X", immediately ask:
- "What other components are already at NODE_X?"
- "Is this a NEW junction or an EXISTING one?"
- "How many things in total connect there? List them explicitly."
- "What does NODE_X DO functionally?" (prevents vague names)

**Why it matters:** Prevents accidental creation of giant catch-all junctions.

## Rule 5: Physical Reference System

For unlabeled components, establish a consistent physical reference.
- Pick: Left/Right, Top/Bottom, or Near/Far
- Use it consistently for the entire board
- Document explicitly: "R5 left pad → X, R5 right pad → Y"

**Why it matters:** Without a consistent reference, ambiguity creeps in. Which pad is which? You'll forget.

## Rule 6: Direction Matters

The DIRECTION of connection matters.
- Series circuit: R1→R2→R3 (each connects left-to-right)
- Parallel circuit: R1 and R2 both connect between same two nodes
- Bias divider: R1 top to power, R2 bottom to ground, junction in middle
- Getting this wrong = completely incorrect netlist

**Why it matters:** This is THE fundamental difference between working and broken circuits.

## Rule 7: DC Path to Ground is Non-Negotiable

Every node MUST have a DC path to ground (node 0/GND).
- If a node seems "floating", add a high-value resistor (1MΩ-1GΩ) to ground
- Capacitors in series = floating node = problem
- This is not optional for SPICE simulation to work

**Why it matters:** Floating nodes cause convergence failures. SPICE requires DC paths.

## Rule 8: Hierarchical Organization Scales

Use subcircuits and multi-file organization for complex boards.
- Don't put everything in one flat file
- Group related components (power stage, gate drive, feedback) in sections
- Use `.include` directives for large projects

**Why it matters:** Flat files become unmanageable. Organization reveals mistakes.

## Rule 9: Names Describe Function, Not Location

Node names should answer "what does this node DO?"

- ❌ Bad: "R1_R2_JUNC" (describes physical connection, not function)
- ✅ Good: "GATE_BIAS" (describes what signal is there)
- ✅ Good: "Q7_DRAIN" (describes component pin being referenced)

**Why it matters:** Functional names make circuits understandable. Location-based names hide meaning.

## Rule 10: Verification Before Finalization

Always read back the netlist before declaring it correct.
- For each component: "This is a [type/value] from [NODE1] to [NODE2]. Correct?"
- For each junction: "This junction has [N] things connected: [list them]. Correct?"
- Does the circuit topology make electrical sense?

**Why it matters:** Catches mistakes before simulation. One verification pass saves hours of debugging.

## Rule 11: Name Everything

This Cadence design rule applies equally to netlists: "Add names on every single transistor, every single inverter, every single net, every single instance."

- ❌ Unlabeled components with default names (R1, C5, M7)
- ✅ Every component has descriptive name (R_Feedback, C_Bypass_Vout, M_MainSwitch)
- ✅ Every net/node has functional name (GATE_BIAS, Q7_DRAIN, CURRENT_SENSE)
- ✅ Hierarchical names enable rapid debugging (Block.Subblock.Signal)

**Why it matters:** Comprehensive labeling is the foundation of professional circuit design. It enables teammates to understand circuit intent instantly, catches errors during review, and supports tracing during simulation debugging.

---

## How These Rules Connect to LLM Research

Recent academic research on LLM-generated netlists (Auto-SPICE, SPICEAssistant, Masala-CHAI) validates these rules:

**Error Prevention:**
- **Floating Nodes** (most common error) → Rule 7 is non-negotiable
- **Connectivity Ambiguity** → Rules 1 & 4 prevent this
- **MOSFET pin confusion** (drain/source swaps) → Rule 5 (physical reference system)
- **Series/parallel errors** → Rule 6 (direction matters)
- **Interpretation failures** → Rule 3 & 11 (functional naming prevents misunderstanding)

**Validation Methods:**
- **Graph Edit Distance (GED):** Structural comparison of netlists catches topology errors
- **Simulation Feedback:** 38% accuracy improvement with iterative refinement (build → test → iterate)
- **5% Tolerance Metrics:** Parameter validation more reliable than exact matches
- **Three-Step Workflow:** Component detection → Prompt engineering → Verification

**Key Insight:** Don't aim for perfect first draft. Use Rules 1-11 as checklist, test with simulation, refine based on results. Research shows this hybrid human-AI approach outperforms pure LLM generation by 46%.

