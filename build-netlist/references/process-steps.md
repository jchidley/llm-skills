# 6-Step Netlist Building Process

Follow this systematic process to build accurate SPICE netlists from circuit descriptions.

## Step 1: Initialize Tracking Table

Create a table to track all connections as the circuit is traced:

| Component | Pad/Pin 1 | Node 1 | Pad/Pin 2 | Node 2 | Status |
|-----------|---|---|---|---|---|
| R1 | Left | +5V | Right | FB_DIV | ✓ |
| R2 | Left | FB_DIV | Right | GND | ✓ |

**Never finalize without both pads documented for every component.**

## Step 2: Ask Three Critical Questions for Each Component

### For Unlabeled Components (R, C, L):
Establish and document physical reference.
- Pick reference convention: Left/Right, Top/Bottom, or Near/Far
- Use consistently for entire board
- Example: "R5 left pad → +5V_RAIL, R5 right pad → GATE_BIAS"

### For Any Component:
1. **"What connects to [side/pin 1]?"** → Document node name.
2. **"What connects to [side/pin 2]?"** → Document node name (or confirm same junction if applicable).
3. **"If different nodes: are they the SAME junction or DIFFERENT junctions?"**

## Step 3: Validate Junctions—Ask "What Else is Here?"

**BEFORE confirming any connection:**
- "What other components already connect to [NODE_NAME]?"
- "Is this a NEW junction or an EXISTING one?"
- "How many components total? List them explicitly."
- "What does [NODE_NAME] DO functionally?" (prevents vague names)

**Why:** Prevents accidental junction bloat (e.g., NODE_A ends up with 10 things).

## Step 4: Document Explicitly

Don't say: "R2 connects to R1"
**Say:** "R2's left pad connects to R1's right pad (both at BIAS_DIV junction)"

**Good example:**
```
R1: Left(+5V) -- Right(BIAS_DIV)
R2: Left(BIAS_DIV) -- Right(GND)
```

**Bad example:**
```
R1 connects to R2
(Which sides? Series or parallel? UNKNOWN!)
```

## Step 5: Name Nodes Functionally

**Never use:** NODE_A, n1, n2, JUNC_1 in final netlist.

**Use instead:** GATE_BIAS, Q1_DRAIN, PWM_FILTERED, CURRENT_SENSE, FEEDBACK_DIV

**Rule:** Each node name answers "what does this signal DO?"

### Naming by Function:
- Voltage dividers: `[SIGNAL]_BIAS` or `[SIGNAL]_DIVIDER`
- Component pins: `[COMPONENT]_[PIN]`
- Power rails: `[VOLTAGE]_RAIL` or specific name like `C1P`
- Filtered signals: `[SIGNAL]_FILTERED`

### Validate Junction Scope:
- 2-4 components per junction = normal
- 5-7 components = suspicious, re-examine
- 8+ components = wrong topology, refactor

## Step 6: Output and Verify

Once all connections documented:

### Read Back Every Component and Junction:
- "R1: 10kΩ from +5V to BIAS_DIV. Correct?"
- "BIAS_DIV junction: R1 right, R2 left, Q1 base. That's 3 connections. Correct?"

### Verify Topology:
- Series chains make sense?
- Parallel branches make sense?
- All nodes have DC paths to GND?

### Verification Techniques (Research-Backed)

**1. Explicit Component Listing:**
For each junction, list ALL components explicitly:
- ✅ Good: "GATE_BIAS has: R1 right pad, R2 left pad, Q1 base. Total: 3 components"
- ❌ Bad: "GATE_BIAS has a few things connected"

**2. Graph-Based Validation (Topology Verification):**

Draw or describe the connection graph explicitly:
- "R1 connects VOUT to GATE_BIAS"
- "R2 connects GATE_BIAS to GND"
- "C1 connects GATE_BIAS to GND (parallel with R2)"

**Graph-Based Validation Checklist:**

This technique, validated by Auto-SPICE research, catches subtle topology errors:

- [ ] **Every node has incoming and outgoing paths** - No isolated components
  - ❌ Wrong: "D1 has cathode at NODE_A, anode at NODE_B, but nothing else connects"
  - ✅ Correct: "D1 freewheels Q7: anode at Q7_DRAIN, cathode at Q7_SOURCE"

- [ ] **Redundant connections identified** - Same nodes listed twice means error
  - ❌ Wrong: GATE_BIAS connects to R1, R2, R1, Q1 (R1 listed twice!)
  - ✅ Correct: GATE_BIAS connects to R1, R2, Q1 (each once)

- [ ] **Series chains verified** - Each intermediate node has exactly 2 connections in chain
  - ❌ Wrong: "VCC--[R1]--NODE_A--[Q1_BASE] and NODE_A also connects to R2 and C1" (4 total—not a series chain)
  - ✅ Correct: "VCC--[R1]--GATE_BIAS--[R2]--GND" (series divider)

- [ ] **Parallel networks identified** - Parallel elements share both endpoints
  - ✅ Correct: "R2 and C1 both connect GATE_BIAS to GND (parallel bypass)"

- [ ] **Power rails connected** - All nodes reachable from VCC/GND
  - ❌ Wrong: NODE_X connects to Q1 base, but no DC path from NODE_X to ground
  - ✅ Correct: All nodes have DC path VCC → (some resistors) → GND

This reveals duplicate connections, unexpected topologies, and floating nodes.

**3. Junction Scope Check:**
- 2-4 components per junction = normal ✅
- 5-7 components = suspicious, re-examine 🤔
- 8+ components = likely wrong, refactor ❌

**4. Functional Naming Validation:**
Can you answer "what does [NODE_NAME] DO?" without referring to the netlist?
- ✅ "GATE_BIAS provides gate voltage through divider"
- ❌ "NODE_A has some stuff connected"

If you can't explain the function, the node is either misnamed or topology is wrong.

**5. Simulation-Based Validation (Optional but Recommended):**
Run QSPICE and check for:
- Convergence errors (often indicate floating nodes or bad connections)
- Extreme voltages/currents (sign of wrong topology)
- Unexpected signal values

Use 5% tolerance range for parameter verification.

### Output Format:

```spice
* ============================================================
* POWER STAGE
* ============================================================
R1 C1P Q7_GATE 10k
C1 Q7_GATE GND 100n
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05
R4 C1P Q7_SOURCE 0.51
```

---

## Quick Questions to Ask When Tracing

1. "Does [COMP] have both ends on SAME junction or DIFFERENT junctions?"
2. "You said connects to A and B. Are A and B the SAME node or DIFFERENT?"
3. "Is this SERIES (chained) or PARALLEL (both to same nodes)?"
4. "Which physical PAD (left/right, top/bottom) connects where?"
5. "[COMP] forms: [NODE1]--[COMP]--[NODE2]. Correct?"
6. "Both pads same node? (Would be 0Ω jumper)"

---

## Pre-Finalization Checklist

- [ ] Every passive component has BOTH pads documented
- [ ] No component has both pads at same node (unless 0Ω jumper)
- [ ] Physical reference consistent (left/right, top/bottom, near/far)
- [ ] Multi-component junctions list all components explicitly
- [ ] Node names describe FUNCTION (GATE_BIAS, not R1_R2_JUNC)
- [ ] 2-4 components per junction typical (8+ = red flag)
- [ ] Every node has DC path to GND
- [ ] Topology verified electrically (series/parallel/bias dividers make sense)
- [ ] All connections read back and confirmed
- [ ] Ready for QSPICE simulation

