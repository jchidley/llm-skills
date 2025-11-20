# Naming Conventions for Netlists

Clear, consistent naming prevents errors and makes netlists self-documenting. These conventions prevent the "giant NODE_A" problem and ensure circuit topology is unambiguous.

## ⚠️ CRITICAL UNIT WARNING

**This is the #1 mistake in SPICE netlists and resistor value specifications:**

### 1M = 1 milliohm, NOT 1 megaohm

The letter "M" in SPICE multipliers means **milli (10^-3)**, not mega.

**Catastrophic Examples:**
- ❌ `R_Feedback VCC FB 1M` → 1 milliohm (nearly a short circuit!)
- ❌ `R_Pull 3V3 EN 10M` → 10 milliohms (major current draw!)
- ✅ `R_Feedback VCC FB 1Meg` → 1 megaohm (correct)
- ✅ `R_Pull 3V3 EN 10Meg` → 10 megaohms (correct)

**Why this happens:** The "M" multiplier predates modern SI prefixes. It was defined as 10^-3 in early SPICE, and changing it would break legacy files.

**Multiplier Table for Reference:**
| Suffix | Name | Value | Example |
|--------|------|-------|---------|
| **T** | Tera | 10^12 | 1T = 1,000,000,000,000 |
| **G** | Giga | 10^9 | 1G = 1,000,000,000 |
| **Meg** | Mega | 10^6 | 1Meg = 1,000,000 ✅ Use this! |
| **K** | Kilo | 10^3 | 1K = 1,000 |
| **m** (none) | — | 10^0 | 1 = 1 |
| **M** | Milli | 10^-3 | 1M = 0.001 ⚠️ Easy mistake! |
| **u** / **μ** | Micro | 10^-6 | 1u = 0.000001 |
| **n** | Nano | 10^-9 | 1n = 0.000000001 |
| **p** | Pico | 10^-12 | 1p = 0.000000000001 |
| **f** | Femto | 10^-15 | 1f = 0.000000000000001 |

**Prevention:**
- **Always spell out "Meg"** when you mean megaohm
- **Never use "1M"** for a high-value resistor—it will be interpreted as 1 milliohm
- **When in doubt**, write out the full number: `1000000` instead of `1Meg`

---

## Component Naming Conventions

All SPICE components MUST start with correct prefix: R, C, D, Q, M, V, I, B, X

| Prefix | Component | Example |
|--------|-----------|---------|
| **R** | Resistor | R_Gate, R_Q7_Gate, R_Sense |
| **C** | Capacitor | C_Bypass, C_Bypass_Vout |
| **D** | Diode | D_Freewheel, D_1N4148 |
| **Q** | BJT Transistor | Q_Driver, Q_2N3906 |
| **M** | MOSFET | M_Q7, M_NMOS, M_PNP |
| **V** | Voltage Source | V_Alt, V_PWM, V_12V |
| **I** | Current Source | I_Load, I_Sense |
| **B** | Behavioral Source | BV_FB, BI_Monitor, BR_Dynamic |
| **X** | Subcircuit | X_Opamp, X_Regulator |

### Component Naming Rules:

- ❌ R1, C5 (meaningless, just numbers)
- ✅ R_Gate, C_Bypass_Vout (self-documenting)
- Include context: R_Q7_Gate (says "gate resistor for Q7")
- Use underscores consistently: R_Gate not RGate
- Avoid: R_1, R_2 (confuses with node numbering)

**Why it matters:** Component names should tell you their PURPOSE instantly. R_Feedback is better than R5 because you know what it does.

## Node (Junction) Naming Conventions

Node names are even more critical because they define circuit topology.

**Never use:** NODE_A, n1, n2, JUNC_1 in final netlist.

**Use instead:** GATE_BIAS, Q1_DRAIN, PWM_FILTERED, CURRENT_SENSE, FEEDBACK_DIV

### Node Naming Rules:

1. **Name after FUNCTION not location**
   - ✅ GATE_BIAS (describes what signal is there)
   - ✅ Q7_DRAIN (describes component pin)
   - ❌ NODE_A (meaningless)
   - ❌ R1_R2_JUNC (describes location, not function)

2. **Functional naming patterns:**
   - Voltage dividers: `[SIGNAL]_BIAS` or `[SIGNAL]_DIVIDER` (e.g., GATE_BIAS, FEEDBACK_DIV)
   - Component pins: `[COMPONENT]_[PIN]` (e.g., Q7_DRAIN, Q7_GATE, Q1_BASE)
   - Power rails: `[VOLTAGE]_RAIL` or explicit name like `C1P`, `VCC_12V`
   - Filtered signals: `[SIGNAL]_FILTERED` (e.g., PWM_FILTERED)
   - Monitoring points: `[SIGNAL]_SENSE` (e.g., CURRENT_SENSE)
   - Output signals: `[SIGNAL]_OUT` (e.g., GATE_DRIVE_OUT)

3. **Power rail naming:**
   - Use explicit names: VCC, VDD, GND, C1P (from schematic)
   - ✅ Good: C1P, VCC_12V, GND
   - ❌ Avoid: POWER, POS, NEG (too vague)

4. **Node name length constraints (HSPICE compatibility):**
   - **Keep names ≤16 characters** for maximum compatibility
   - Names longer than 16 characters may be replaced with numbers by some simulators
   - **Examples:**
     - ✅ `GATE_BIAS` (10 chars) - safe
     - ✅ `Q7_DRAIN` (8 chars) - safe
     - ✅ `FB_DIV_OUT` (10 chars) - safe
     - ⚠️ `FEEDBACK_DIVIDER_OUTPUT` (24 chars) - may be truncated
     - ⚠️ `GATE_DRIVE_CIRCUIT_IN_1` (24 chars) - may be truncated
   - **When names are too long, abbreviate:**
     - ❌ `GATE_DRIVER_BOOTSTRAP_CAPACITOR_VOLTAGE`
     - ✅ `BOOT_CAP_RAIL` (13 chars)
   - **Test:** If you can't read your node name comfortably without squinting, it's probably too long

5. **Ground node:**
   - SPICE requires ground to be `0` or `GND`
   - Use `GND` consistently (more readable)

6. **No-connect (NC) nodes:**
   - When a component terminal is intentionally not connected (NC pin, unused pad)
   - Use unique functional names: `NC_[COMPONENT]_[PAD]`
   - Examples:
     - ✅ `NC_R5_PAD2` (R5's second pad is NC)
     - ✅ `NC_U1_PIN7` (U1's pin 7 is NC)
     - ✅ `NC_IC_TESTPOINT` (test point deliberately unconnected)
   - **Important distinction:**
     - `NC_[name]` = **deliberately** not connected (by design)
     - `FLOATING_[name]` = **accidentally** not connected (error—fix it)
   - Never use generic NC names like `NC`, `NC1`, `NC2` (defeats the purpose)

### Validate Junction Scope:

- **2-4 components** at a junction = probably correct
- **5-7 components** = suspicious, re-examine topology
- **8+ components** = likely wrong, refactor with intermediate nodes

**Why it matters:** Vague node names hide topology errors. If GATE_BIAS has 10 things connected, it's wrong.

---

## The "Giant NODE_A" Problem

This is the #1 netlist mistake. Here's how it happens and how to prevent it:

### Bad Approach (Junction Bloat):
```spice
R1 VCC NODE_A 10k
R2 NODE_A GND 22k
C1 NODE_A GND 10u
Q1 NODE_A BASE NODE_A 2N3906    ← Wait, Q1 collector at NODE_A?
D1 NODE_A GND D_1N4148          ← D1 also here?
BV_Feedback FB GND V=V(NODE_A)  ← Feedback from NODE_A?
I_Monitor NODE_A GND I=10m      ← Monitoring NODE_A?

* Now NODE_A has 7 things. Is this right? IMPOSSIBLE TO TELL!
```

### Good Approach (Functional Naming):
```spice
R1 VCC GATE_BIAS 10k             ← Clear: bias divider
R2 GATE_BIAS GND 22k
C1 GATE_BIAS GND 10u              ← Bypass cap

Q1 Q1_DRAIN GATE_BIAS GND Q_2N3906  ← Clear: Q1 base at bias
D1 Q1_DRAIN GND D_1N4148            ← D1 freewheels Q1

BV_Feedback FB GND V=V(GATE_BIAS)/2  ← Feedback measures bias
I_Monitor GATE_BIAS GND I=I(R2)       ← Monitor through R2

* Each node has PURPOSE and SMALL number of components
```

### How to Prevent Junction Bloat:

When user says "connects to [NODE_NAME]", ask immediately:
1. "What other components are already at [NODE_NAME]?"
2. "Is this a NEW junction or EXISTING?"
3. "How many components total?"
4. "What does [NODE_NAME] DO functionally?"

Then name accordingly:
- If 2-3 components at divider output: BIAS_DIVIDER or GATE_BIAS
- If single pin reference: Q7_GATE or Q1_DRAIN
- If power distribution: C1P or VCC_12V
- If filtering: PWM_FILTERED
- If monitoring: CURRENT_SENSE

**Why:** Naming forces you to understand topology. If you can't name it functionally, you don't understand what it does.

---

## Node Naming Anti-Patterns

**Never use these in a final netlist:**

| Name | Problem | Better Name |
|------|---------|---|
| `NODE_A`, `NODE_B` | No context | Use functional name |
| `n1`, `n2`, `n3` | Temporary only | Use functional name |
| `JUNC_1`, `JUNC_2` | No meaning | Describe what's there |
| `NET_1`, `NET_2` | Too vague | Name the signal/function |
| `SIGNAL` | Which signal? | `PWM_SIGNAL`, `FEEDBACK` |
| `OUTPUT` | Output of what? | `GATE_DRIVE_OUT`, `SENSE_OUT` |
| `INPUT` | Input to what? | `PWM_INPUT`, `ENABLE_INPUT` |
| `POWER` | What voltage? | `VCC_12V`, `GATE_RAIL_10V` |
| `GROUND` | Always GND | Use `GND`, never `GROUND` |
| `NC`, `NC1`, `NC2` | Too generic, loses info | `NC_R5_PAD2`, `NC_U1_PIN7` |
| `FLOATING` | Vague, suggests error | Clarify: is it intentional NC or accidental? |

---

## Real Example: Life Fitness Board

**Bad approach (junction bloat):**
```spice
R14 VCC NODE_A 127
U1 NODE_A NODE_A NODE_A ...  ← What?? NODE_A for feedback?
```

**Good approach (functional naming):**
```spice
* R14 is part of LM1086 regulator feedback divider
* Divider sets output to ~6V (1.25V ref * (1 + R14/R_GND))
R14 U1_OUT U1_FB 127

.subckt LM1086 IN FB OUT
  ...
.ends

X_U1 C1P U1_FB U1_OUT LM1086
```

Much clearer! U1_OUT and U1_FB are specific, functional nodes.

---

## Validation Questions for Node Naming

Before accepting a node name, ask:

- [ ] Can someone unfamiliar with the circuit understand what this node does from the name?
- [ ] Does the node name describe the FUNCTION (divider, bias, feedback, rail, sense, etc.)?
- [ ] Have I explicitly listed all components connected to this node?
- [ ] Does the number of components make sense (2-4 typical, not 8+)?
- [ ] Would I find this node in a schematic based on the name?
- [ ] Is the node name specific (not generic like NODE_A)?
- [ ] If I explained this junction to someone else, would they immediately understand it?

If answer is "no" to any of these, rename the junction to be more specific.

