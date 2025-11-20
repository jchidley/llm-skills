# NC Naming Convention - Before/After Examples

## Example 1: Unpopulated Resistor Footprint

**Before (unclear):**
```spice
R4 VCC ??? 10k
* What's the right pad? Is this intentional or did we miss documentation?
```

**After (clear):**
```spice
R34 VCC NC_R34_PAD2 10k
* Right pad is deliberately unconnected (unpopulated resistor)
```

---

## Example 2: IC Pin Marked as No-Connect

**Before (ambiguous):**
```spice
U1 IN FB NC 0 ... ...
* Is that third terminal really unconnected, or did we miss it?
```

**After (explicit):**
```spice
U1 IN FB NC_U1_PIN7 0 ... ...
* Pin 7 marked NC on datasheet
```

---

## Example 3: Test Point with Only One Pad Soldered

**Before (confusing):**
```spice
TP1 ???
* How many pads? Are both connected? One floating?
```

**After (clear):**
```spice
TP1 GATE_BIAS NC_TP1_PAD2
* TP1 pad 1 = gate bias signal
* TP1 pad 2 = deliberately unconnected (single-pad test point for measurement)
```

---

## Example 4: BJT with One Floating Pin (Error Case)

**During tracing (temporary):**
```spice
Q12 Q12_COLL Q12_BASE FLOATING_Q12_EMITTER 2N3906
* Q12 emitter - need to trace this!
```

**After verification (resolved):**
```spice
Q12 Q12_COLL Q12_BASE GND 2N3906
* Q12 emitter tied to ground
```

OR if it's truly unconnected by design:

```spice
Q12 Q12_COLL Q12_BASE NC_Q12_EMITTER 2N3906
* Q12 emitter deliberately NC (confirms it's high-impedance input)
```

---

## Example 5: MOSFET with All Terminals Documented

**Before (mixed clarity):**
```spice
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE ???
* Bulk/substrate - where is it?
```

**After (complete):**
```spice
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05
* Bulk tied to source (standard for single MOSFET)
```

OR if bulk is intentionally floating:

```spice
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE NC_Q7_BULK NMOS_2N03L05
* Bulk intentionally unconnected (unusual but documented)
```

---

## Example 6: Multi-Pin IC with Mixed Connections

**Before (unclear which pins matter):**
```spice
X_U2 + - OUT VS CC GND LM393P
* What's on VS? What's on CC? Are these really unconnected?
```

**After (explicit):**
```spice
X_U2 P1_PIN5 P1_PIN6 COMPARATOR_OUT VCC_12V NC_U2_PIN8 GND LM393P
* P1_PIN5 = + input
* P1_PIN6 = - input
* COMPARATOR_OUT = output
* VCC_12V = positive supply
* NC_U2_PIN8 = pin 8 unconnected (per datasheet)
* GND = ground reference
```

---

## Example 7: Populated Resistor Network

**Full Documentation:**
```spice
* Voltage divider with bypass capacitor
R1 C1P BIAS_DIV 100k          ; Top resistor, documented
R2 BIAS_DIV GND 22k           ; Bottom resistor, documented
C1 BIAS_DIV GND 100n          ; Bypass capacitor, documented

* Other resistor: unpopulated on this board
R3 FEEDBACK NC_R3_PAD2 10k    ; Left pad = feedback signal
                               ; Right pad = deliberately NC
```

---

## Key Patterns

| Situation | Before | After |
|-----------|--------|-------|
| Unpopulated footprint | `???` (ambiguous) | `NC_R5_PAD2` (clear) |
| NC pin on IC | `NC` (generic) | `NC_U1_PIN7` (specific) |
| Floating pad during trace | Undocumented | `FLOATING_Q12_EMIT` (temporary) |
| Verified as intentional NC | `NC` or undocumented | `NC_[COMPONENT]_[PAD]` (permanent) |
| Bulk/substrate connection | Unclear | `NC_M1_BULK` or `M1_SOURCE` (explicit) |
| Test point, one pad used | One pad documented, other ignored | `TP1_SIGNAL NC_TP1_PAD2` (both documented) |

---

## Validation Checklist

Before finalizing a netlist:

- [ ] Every component is listed
- [ ] Every terminal is documented (connected node, NC with unique name, or power/ground)
- [ ] No generic NC names (verify all NC nodes have `NC_[COMPONENT]_[PAD]` format)
- [ ] No `FLOATING_[name]` in final netlist (all floating nodes resolved or marked as intentional NC)
- [ ] Every two-terminal component: both pads documented
- [ ] Every three-terminal component (BJT): all three pins documented
- [ ] Every four-terminal component (MOSFET): D, G, S, B all documented
- [ ] Every N-terminal component (IC): all N pins documented
- [ ] NC naming is functional: `NC_R34_PAD2` not `NC_THING`

---

## Why This Matters for SPICE Simulation

**Floating nodes cause:**
- Convergence errors
- Undefined voltages
- Meaningless simulation results
- Difficult debugging

**Explicit documentation (including NC):**
- ✅ SPICE converges (all nodes have defined paths or are intentionally NC)
- ✅ Netlist is self-documenting (you can read it without diagram)
- ✅ Errors are visible (obvious if something is missing)
- ✅ Design intent is captured (why something is NC)

**Example: SPICE convergence failure vs success**

**Fails to converge:**
```spice
Q1 Q1_COLL BIAS_POINT ???      ; Emitter unspecified!
```

**Converges, simulates correctly:**
```spice
Q1 Q1_COLL BIAS_POINT GND Q_2N3906     ; Emitter to ground
R_Load Q1_COLL VCC 1k           ; DC path to ground via R_Load
```

**Converges, correct for high-impedance input:**
```spice
Q1 Q1_COLL BIAS_POINT NC_Q1_EMIT Q_2N3906   ; Emitter intentionally NC
R_Load Q1_COLL VCC 1k                        ; DC path through R_Load
```

The difference: **explicit documentation** makes simulation work and design intent clear.
