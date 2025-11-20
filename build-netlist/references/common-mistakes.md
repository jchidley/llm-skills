# Common Netlist Mistakes & How to Fix Them

## Component Connection Mistakes

### Mistake 1: Half-Traced or Undocumented Component Terminals

**What it looks like:**
```
R5 connects to +5V
(What about the other pad?)

OR

U1 pin 7 - ???
(Is it NC? Floating? Grounded?)
```

**Why it's wrong:** Missing terminal documentation creates floating nodes, which cause SPICE convergence failures.

**Fix:**
```
R5 left pad → +5V_RAIL
R5 right pad → GATE_BIAS

U1 pin 7 → NC_U1_PIN7  (deliberately unconnected)
```

**Key rule:** Every terminal of every component must be documented. For each terminal:
1. Is it connected to a signal node?
2. Is it deliberately NC (no-connect)? → Use unique name `NC_[COMPONENT]_[PAD]`
3. Is it tied to power/ground?
4. **Never leave a terminal undocumented.**

---

### Mistake 2: Ambiguous Side References

**What it looks like:**
```
R1 connects to R2
R2 connects to R3
(Which sides? Series or parallel? UNKNOWN!)
```

**Why it's wrong:** No way to know if components are in series, parallel, or mixed.

**Fix:**
```
R1: Left(+5V) -- Right(BIAS_DIV)
R2: Left(BIAS_DIV) -- Right(GND)
R3: Left(Q7_GATE) -- Right(GND)
```

**Key rule:** Always specify which physical side connects where.

---

### Mistake 3: Both Pads Same Node

**What it looks like:**
```
R28 JUNC_B JUNC_B 5.2k
(Resistor connects node to itself?)
```

**Why it's wrong:** Physically impossible unless it's a 0Ω jumper. Indicates topology misunderstanding.

**Fix:**
- Verify the two pads actually connect to different nodes
- Trace both sides again
- Ask: "Which side connects to [JUNC_B]? What about the other side?"

**Key rule:** If both pads are at the same node, one must be wrong.

---

## Junction Naming Mistakes

### Mistake 4: Generic Node Names (The Giant NODE_A Problem)

**What it looks like:**
```spice
R1 VCC NODE_A 10k
R2 NODE_A GND 22k
C1 NODE_A GND 10u
Q1 NODE_A BASE NODE_A 2N3906    ← Q1 collector at NODE_A?
D1 NODE_A GND D_1N4148          ← D1 also here?
BV_Feedback FB GND V=V(NODE_A)  ← Monitoring NODE_A?

* Now NODE_A has 7 things connected. Is this right? IMPOSSIBLE TO TELL!
```

**Why it's wrong:** Vague names hide topology errors. You can't tell if this is correct.

**Fix:**
```spice
R1 VCC GATE_BIAS 10k             ← Clear: bias divider
R2 GATE_BIAS GND 22k
C1 GATE_BIAS GND 10u

Q1 Q1_DRAIN GATE_BIAS GND Q_2N3906  ← Q1 base at bias
D1 Q1_DRAIN GND D_1N4148            ← D1 freewheels Q1

BV_Feedback FB GND V=V(GATE_BIAS)/2
```

**Key rule:** 2-4 components per junction = normal. 8+ = wrong topology.

**How to prevent:** When user says "connects to [NODE]", immediately ask:
- "What other components already connect there?"
- "How many total?"
- "What does this node DO?"

---

### Mistake 5: Location-Based Names Instead of Function-Based

**What it looks like:**
```
R1_R2_JUNC (describes physical connection, not function)
R1_R2_R3_JUNCTION (worse)
```

**Why it's wrong:** Doesn't explain what signal is there. Hides circuit understanding.

**Fix:**
```
GATE_BIAS (describes what signal is there)
Q7_GATE (describes component pin)
PWM_FILTERED (describes what processing is applied)
```

**Key rule:** Node names should answer "what does this signal DO?"

---

## Topology Mistakes

### Mistake 6: Series/Parallel Confusion

**What it looks like:**
```
"R1 and R2 are both connected to something"
(Are they in series or parallel?)
```

**Why it's wrong:** These are completely different circuits with different behavior.

**Series:** R_total = R1 + R2
```
+5V --[R1]--[R2]--GND
```

**Parallel:** 1/R_total = 1/R1 + 1/R2
```
+5V --[R1]--+5V
    --[R2]--+5V
```

**Fix:**
- Trace BOTH pads of each component
- Verify which nodes they connect to
- Confirm they share nodes or don't

---

### Mistake 7: Missing DC Path to Ground

**What it looks like:**
```
R1 VCC NODE_A 10k
C1 NODE_A NODE_B 100n
(Node B has no path to ground!)
```

**Why it's wrong:** SPICE requires DC paths to ground for every node.

**Fix:**
```
R1 VCC NODE_A 10k
C1 NODE_A NODE_B 100n
R2 NODE_B GND 1Meg        ← Add DC path (high impedance)
```

**Key rule:** Every node MUST have a DC path to GND. This is non-negotiable for simulation.

---

### Mistake 8: Wrong Bias Divider Understanding

**What it looks like:**
```
R1 VCC NODE_A 10k
R2 NODE_A GND 22k
* What is the voltage at NODE_A?
* (User doesn't know)
```

**Why it's wrong:** Bias dividers have a specific function—producing a middle voltage.

**Fix:**
```
* Voltage divider from VCC to GND
* Output at middle = VCC * R2/(R1+R2)
R1 VCC BIAS_DIV 10k         ← Top resistor
R2 BIAS_DIV GND 22k         ← Bottom resistor
C1 BIAS_DIV GND 100n        ← Bypass capacitor (optional)

* With R1=10k, R2=22k: V(BIAS_DIV) = VCC * 22/(10+22) = 0.69*VCC
```

**Key rule:** Understand what EACH junction does before naming it.

---

## Component-Specific Mistakes

### Mistake 9: MOSFET Pin Confusion

**What it looks like:**
```
M_Q7 SOURCE GATE DRAIN Q7_SOURCE NMOS
(Wrong order! MOSFET format is D G S B)
```

**Why it's wrong:** Wrong pin order creates completely broken circuit.

**Fix:**
```
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05
      ↑        ↑        ↑        ↑
      D        G        S        B (bulk)
```

**Key rule:** Remember SPICE order for transistors:
- BJT: `Q_name COLLECTOR BASE EMITTER MODEL`
- MOSFET: `M_name DRAIN GATE SOURCE BULK MODEL`

---

### Mistake 10: Capacitor Polarity Ignored

**What it looks like:**
```
C_IN_POS IN GND 10u
(Electrolytic cap connected backwards?)
```

**Why it's wrong:** Polarized capacitors only work in one direction.

**Fix:**
- Note polarity in comments
- Verify positive side connects to higher voltage
- Example: Positive to signal, negative to ground for input coupling

```spice
* Coupling capacitor - positive to signal, negative to bias point
C_IN SIGNAL BIAS_POINT 10u
```

**Key rule:** Document polarized components. Add comments showing polarity.

---

## Naming Mistakes

### Mistake 11: Generic Signal Names

**What it looks like:**
```
V_INPUT IN GND ...
V_OUTPUT OUT GND ...
BV_SIGNAL SIG GND ...
(What INPUT? What OUTPUT? For what?)
```

**Why it's wrong:** Too vague. Doesn't describe what the signal is.

**Fix:**
```
V_PWM PWM_INPUT GND ...
V_FB FEEDBACK_OUTPUT GND ...
BV_GATE_CLAMP GATE_CLAMP GND ...
```

**Key rule:** Make signal names specific and functional.

---

### Mistake 12: Mixed Case Inconsistency

**What it looks like:**
```
R1 VCC Gate_Bias 10k
R2 gate_bias GND 22k
C1 GATE_BIAS GND 10u
(Gate_Bias, gate_bias, GATE_BIAS - three different names!)
```

**Why it's wrong:** SPICE is case-sensitive. These are three different nodes!

**Fix:**
```
R1 VCC GATE_BIAS 10k
R2 GATE_BIAS GND 22k
C1 GATE_BIAS GND 10u
```

**Key rule:** Pick a case convention and stick to it. ALL_CAPS is typical for node names.

---

## LLM-Generated Netlist Mistakes

When using LLMs to help build netlists, watch for these AI-specific errors (backed by Auto-SPICE and SPICEAssistant research):

### Mistake 13: MOSFET Drain/Source Terminal Swapping

**What it looks like:**
```spice
M_Q7 Q7_SOURCE Q7_GATE Q7_DRAIN Q7_DRAIN NMOS
     ↑ SOURCE    GATE    ↑ DRAIN
     (WRONG ORDER!)
```

**Why LLMs do this:** Difficulty distinguishing gate vs drain in dense layouts or schematics. Often happens with symmetrical transistor symbols.

**Fix - Always verify pin order:**
```spice
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05
     ↑        ↑        ↑        ↑
     D        G        S        B (correct!)
```

**Prevention:** Trace each transistor pin explicitly. Ask LLM: "Which pin is DRAIN? Which is SOURCE? Which is GATE?" Get explicit confirmation before accepting.

---

### Mistake 14: Incorrect Connection Assumptions at Non-Junctions

**What it looks like:**
```
LLM assumes intersecting wires/traces are connected when they shouldn't be.
Example: Two wires cross in schematic but don't have a dot junction indicator.
```

**Why LLMs do this:** Visual interpretation ambiguity. Crossing vs junction dots not always clear in image/schematic format.

**Fix:**
- **Always ask:** "Does [SIGNAL1] physically touch [SIGNAL2], or do they just cross?"
- Verify with schematic visual inspection
- Check netlist connectivity for unintended junctions

**Prevention:** Use behavioral descriptions to force explicit connectivity statements. "List every signal that physically touches [NODE_NAME]" instead of "trace the circuit".

---

### Mistake 15: Differential Pair Input/Output Mishandling

**What it looks like:**
```spice
* Swapped input/output in differential pair
BI_DIFF SENSE GND I=V(OUTPUT)   ← Should be INPUT!
```

**Why LLMs do this:** Differential circuit symmetry confuses signal direction inference.

**Fix:**
- Explicitly name signals: `DIFF_INPUT_PLUS`, `DIFF_INPUT_MINUS`, `DIFF_OUT`
- Ask LLM to trace INPUT signals separately from OUTPUT signals
- Verify direction against schematic/function

**Prevention:** Use ultra-specific naming. "CURRENT_SENSE_IN" not "SENSE", "FB_OUTPUT" not "FB".

---

### Mistake 16: Unit Suffix Errors (1M vs 1Meg)

**What it looks like:**
```spice
R_FEEDBACK VCC FB 1M    ← LLM assumes "M" means Mega!
(Actually 1 milliohm—catastrophic error!)
```

**Why LLMs do this:** Training data doesn't emphasize this historical SPICE quirk.

**Fix:** Always expand unit suffixes explicitly
```spice
R_FEEDBACK VCC FB 1Meg    ← Correct (or spell out: 1000000)
```

**Prevention:** Add comment showing actual resistance:
```spice
R_FEEDBACK VCC FB 1Meg   * 1,000,000 ohm feedback resistor
```

**Critical:** Always verify high-value resistors use "Meg" not "M". This is the #1 unit mistake in SPICE netlists.

---

### Mistake 17: Missing Components in Dense Layouts

**What it looks like:**
```
LLM traces main components but misses small components (filter caps, bypass caps, protection diodes).
```

**Why LLMs do this:** Small components near pads may be hard to see in images. Dense layouts with many components.

**Fix:**
- Use checklist verification: "List EVERY component on the board"
- Require explicit count: "I count [N] resistors, [M] capacitors, [K] transistors..."
- Ask: "What's protecting [NODE] from overvoltage?"

**Prevention:** Ask explicit follow-up questions about protection (diodes, caps) rather than letting LLM trace freely.

---

### Mistake 18: Space Characters in Node Names

**What it looks like:**
```spice
R1 VCC RESET 0 10k
R2 RESET0 GND 22k
* Looks like a connection, but...
* "RESET 0" ≠ "RESET0" - These are TWO DIFFERENT nodes!
```

**Why it's wrong:** Space characters create node names with embedded spaces. SPICE interprets spaces as delimiters, not part of the name. So "RESET 0" becomes two separate tokens: node name "RESET" followed by a stray "0".

**Real-world example:**
```spice
V1 VCC GND DC 12
R1 VCC A 0 10k   ← "A 0" is "node A" followed by rogue "0"
R2 A0 GND 22k    ← "A0" is correct single node
* R1 and R2 are NOT connected! Different nodes!
```

**Fix - Be meticulous with naming:**
```spice
* Option 1: Use underscores
R1 VCC RESET_0 10k
R2 RESET_0 GND 22k

* Option 2: No separators
R1 VCC RESET0 10k
R2 RESET0 GND 22k

* Option 3: More descriptive
R1 VCC RESET_LOW 10k
R2 RESET_LOW GND 22k
```

**Key rule:**
- ✅ Valid node names: `RESET_0`, `RESET0`, `RESET_LOW`, `V_OUT`, `Q7_GATE`
- ❌ Invalid node names: `RESET 0` (space), `RESET 0V` (multiple words), `node 1` (space)

**Prevention:**
- Avoid spaces entirely in node names
- Use underscores as separators
- Use hyphens only if your SPICE variant supports them (check documentation)
- When copy-pasting node names, verify no trailing/leading spaces
- Use consistent delimiters: pick either underscores or no separators and stick with it

**Why this matters for reversing:** When migrating schematics or netlist documentation to SPICE, node labels from schematics may have spaces (e.g., "POWER 5V"). Always convert these to `POWER_5V` or `POWER5V` before entering the netlist.

---

## Quick Reference: Mistake Prevention Checklist

| Area | Mistake | Prevention |
|------|---------|-----------|
| **Components** | Half-traced | Document BOTH pads for every component |
| **Components** | Ambiguous sides | Specify "left pad", "right pad", "top", "bottom" |
| **Junctions** | Generic names (NODE_A) | Use functional names (GATE_BIAS, Q7_DRAIN) |
| **Junctions** | Too many components | If 8+ at one node, split into sub-junctions |
| **Junctions** | Location-based names | Names should answer "what does this DO?" |
| **Topology** | Series/parallel confusion | Trace both pads to confirm topology |
| **Topology** | Floating nodes | Verify every node has DC path to GND |
| **Topology** | Wrong divider setup | Understand what each junction does |
| **Components** | MOSFET pin order | Remember: D-G-S-B order |
| **Components** | Capacitor polarity | Document and verify polarized caps |
| **Naming** | Generic signals | Make signal names specific and functional |
| **Naming** | Case inconsistency | Use consistent case (ALL_CAPS typical) |
| **Naming** | Space characters in names | Avoid spaces: use underscores (RESET_0 not RESET 0) |

---

## Validation Before Finalizing

Before considering a netlist complete, ask these questions:

1. "Does every passive component have BOTH pads documented?"
2. "Are node names functional, not generic (GATE_BIAS, not NODE_A)?"
3. "Does each junction have 2-4 components (not 8+)?"
4. "Does every node have a DC path to GND?"
5. "Do I understand what each junction DOES?"
6. "Can I read the netlist and understand the circuit without a diagram?"
7. "Does the topology make electrical sense (series chains, parallel networks, dividers)?"

If answer is "no" to any question, fix it before simulating.

