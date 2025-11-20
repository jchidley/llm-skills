# Example SPICE Netlists

Working examples from real circuit tracing.

## Example 1: Simple Voltage Divider with Bypass

This is a basic gate bias network.

**Circuit Description:**
- Power supply 12V connected to +5V rail
- Two resistors form a divider from +5V to GND, creating a ~3.3V bias point
- Capacitor bypasses the bias point to GND for AC signals
- Output bias drives a transistor base

**Tracing Process:**

```
Component R1 (10kΩ):
- Left pad → +5V_RAIL
- Right pad → GATE_BIAS

Component R2 (22kΩ):
- Left pad → GATE_BIAS (same as R1 right)
- Right pad → GND

Component C1 (100nF):
- Top pad → GATE_BIAS
- Bottom pad → GND
```

**SPICE Netlist:**

```spice
* Gate Bias Network for Q1
* Divider creates ~3.3V at GATE_BIAS

V_PWR C1P GND DC 5                * 5V power supply
R1 C1P GATE_BIAS 10k              * Top of divider: 10kΩ to +5V
R2 GATE_BIAS GND 22k              * Bottom of divider: 22kΩ to GND
C1 GATE_BIAS GND 100n             * Bypass capacitor

* GATE_BIAS voltage = 5V * (22k / (10k + 22k)) = 3.33V
```

---

## Example 2: MOSFET Switching Stage

More complex: power MOSFET with load resistor, freewheeling diode, and gate drive.

**Circuit Description:**
- 12V alternator input at C1P
- Load resistor (0.51Ω shunt) from C1P to Q7 source
- MOSFET Q7 controlled by gate signal
- Freewheeling diode D3 prevents overvoltage on Q7 drain
- PWM input controls gate

**Tracing Process:**

```
Component R4 (0.51Ω load resistor):
- Left pad → C1P (alternator input)
- Right pad → Q7_SOURCE (MOSFET source pin)

Component M_Q7 (NMOS MOSFET):
- Drain pin → Q7_DRAIN
- Gate pin → Q7_GATE
- Source pin → Q7_SOURCE (same as R4 right)
- Bulk → Q7_SOURCE (substrate connection)

Component D3 (1N4148 freewheeling diode):
- Anode → Q7_DRAIN (same as Q7 drain)
- Cathode → Q7_SOURCE (same as Q7 source and R4 right)
```

**SPICE Netlist:**

```spice
* ============================================================
* POWER STAGE - MOSFET Q7 with load resistor
* ============================================================

V_ALT C1P GND DC 12               * Alternator 12V input
V_PWM Q7_GATE GND PULSE(0 10 0 10n 10n 2.5u 5u)

* Load resistor - shunt across Q7 source
* At 20A current: V(R4) = 20A * 0.51Ω = 10.2V (current sense)
R4 C1P Q7_SOURCE 0.51

* Main switching MOSFET
* Drain: Q7_DRAIN (to diode anode and rest of circuit)
* Gate: Q7_GATE (from PWM control)
* Source: Q7_SOURCE (to R4 and D3 cathode)
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05

* Freewheeling diode - prevents inductive spikes
* Anode to Q7 drain (high side)
* Cathode to Q7 source (low side)
D3 Q7_DRAIN Q7_SOURCE D_1N4148

* Transient analysis: 100ms, 1µs max step
.tran 0 100m 0 1u

* Model definitions
.model NMOS_2N03L05 NMOS (KP=2.5m VTO=2.0 LAMBDA=0.02)
.model D_1N4148 D (RS=0.5 CJO=4p VJ=0.8 BV=75 IBV=100u)

.end
```

---

## Example 3: Feedback Network with Behavioral Source

Uses QSPICE behavioral source to simplify feedback divider.

**Circuit Description:**
- Output voltage C1P is monitored
- Voltage divider (1/10 scaling) reduces it for feedback circuit
- Behavioral source replaces physical resistor divider (token-efficient)
- Result fed to comparator

**SPICE Netlist:**

```spice
* ============================================================
* FEEDBACK NETWORK - voltage monitoring and scaling
* ============================================================

* Output voltage source (from previous stage)
V_OUT C1P GND DC 12

* Physical divider approach (would need R1, R2):
* R1 C1P FB_DIV 9k
* R2 FB_DIV GND 1k
* This creates: V(FB_DIV) = V(C1P) * (1k / 10k) = 1/10 scaling

* QSPICE behavioral source approach (more efficient):
* Implements divider mathematically without resistors
BV_FEEDBACK FB_DIV GND V=V(C1P)/10

* Feedback goes to comparator input
V_THRESH THRESH GND DC 1.25      * Reference voltage for comparator
* (Comparator model would follow)

* Analysis
.tran 0 10m 0 100u

.end
```

---

## Example 4: Complete Gate Drive Circuit

Combines divider, bypass, gate resistor, and MOSFET.

**Circuit Description:**
- Bias divider at GATE_BIAS (3.3V)
- Bypass capacitor for noise rejection
- Gate resistor limits switching speed
- MOSFET Q1 controlled by divider output
- Load on Q1 collector

**Tracing Method:**

```
Step 1: Identify all junctions
- GATE_BIAS: divider output, bypass cap, gate resistor connection point
- Q1_COLL: Q1 collector, load resistor, measurement point

Step 2: Trace each component
- R_DIVIDER_TOP: +5V to GATE_BIAS
- R_DIVIDER_BOT: GATE_BIAS to GND
- C_BYPASS: GATE_BIAS to GND (parallel with R_DIVIDER_BOT)
- R_GATE: GATE_BIAS to Q1_BASE (limits current into base)
- Q1_BJTNPN: Collector=Q1_COLL, Base=Q1_BASE, Emitter=GND
- R_LOAD: +5V to Q1_COLL (load resistor)
- R_MEASURE: Q1_COLL to GND via oscilloscope (not in netlist)

Step 3: Verify connections
- GATE_BIAS junction: 3 components (R_divider_top right, R_divider_bot left, R_gate left, C_bypass top)
- Q1_BASE junction: 2 components (R_gate right, Q1 base pin)
- Q1_COLL junction: 2 components (Q1 collector pin, R_load left)
```

**SPICE Netlist:**

```spice
* ============================================================
* COMPLETE GATE DRIVE - Bias divider with BJT driver
* ============================================================

* Power supplies
V_VCC C1P GND DC 5

* Bias divider network
* Creates GATE_BIAS = 5V * (22k / (10k + 22k)) = 3.33V
R_DIVIDER_TOP C1P GATE_BIAS 10k         * Top resistor: 10kΩ
R_DIVIDER_BOT GATE_BIAS GND 22k         * Bottom resistor: 22kΩ
C_BYPASS GATE_BIAS GND 100n              * Bypass: 100nF to GND

* Gate drive to transistor
* R_GATE limits base current: I_base = (V_GATE_BIAS - V_BE) / R_GATE
*                            = (3.33 - 0.7) / 1k ≈ 2.6 mA
R_GATE GATE_BIAS Q1_BASE 1k             * Gate resistor: 1kΩ

* NPN BJT driver transistor
* Q1 base at GATE_BIAS node (through R_GATE)
* Q1 collector drives load
Q1 Q1_COLL Q1_BASE GND Q_2N3904

* Load resistor on collector
R_LOAD C1P Q1_COLL 1k                   * 1kΩ collector load

* When Q1 is ON: V(Q1_COLL) ≈ 0.2V (V_SAT)
* When Q1 is OFF: V(Q1_COLL) ≈ 5V

* Transient analysis
.tran 0 10m 0 100u

* Model
.model Q_2N3904 NPN (BF=200 VAF=100 TF=0.5n)

.end
```

---

## Example 5: Using Your Build Process

This shows how to apply the 6-step process to a real circuit.

**Scenario:** Tracing a regulator circuit from board description

**Step 1 - Initialize Table:**

| Component | Pad 1 | Node 1 | Pad 2 | Node 2 | Status |
|-----------|-------|--------|-------|--------|--------|
| R14 | Left | ? | Right | ? | Pending |
| C1 | Top | ? | Bot | ? | Pending |
| U1 | ? | ? | ? | ? | Pending |

**Step 2 - Ask Questions:**

*For R14:*
- "Left pad connects to... the alternator positive (C1P)"
- "Right pad connects to... U1's feedback pin area"
- "Same junction or different? Different—let's call it U1_FB"

*For C1:*
- "Top pad connects to... same alternator positive (C1P)"
- "Bottom pad connects to... ground rail (GND)"

**Step 3 - Validate Junctions:**

- C1P junction: R14 left, C1 top, U1 input, D1 anode → "Main input rail"
- U1_FB junction: R14 right, R15 left, U1 feedback pin → "Feedback divider output"

**Step 4 - Document:**

```
R14: Left(C1P) -- Right(U1_FB)
C1: Top(C1P) -- Bottom(GND)
```

**Step 5 - Name Nodes:**

- C1P = main alternator input (already named on schematic)
- U1_FB = regulator feedback divider output

**Step 6 - Output:**

```spice
* Regulator input stage
V_ALT C1P GND DC 12
R14 C1P U1_FB 127       * Feedback divider top
R15 U1_FB GND 1k        * Feedback divider bottom
C1 C1P GND 10u          * Input filter

* Regulator IC
* IN: C1P, FB: U1_FB, OUT: U1_OUT, GND: GND
X_U1 C1P U1_FB U1_OUT GND LM1086
```

---

## Quick Netlist Template

Use this structure for all your netlists:

```spice
* ============================================================
* [Circuit Description]
* [Your Name] - [Date]
* ============================================================

* Section 1: Power Supplies
V_supply_name NODE+ NODE- DC/AC value

* Section 2: Main Circuit (organize by functional block)
R_name NODE1 NODE2 value
C_name NODE1 NODE2 value
Q_name PINS... model
M_name PINS... model
D_name ANODE CATHODE model

* Section 3: Analysis Directives
.tran 0 time 0 max_step
.ac dec pts_per_decade start_freq stop_freq

* Section 4: Models
.model name type (parameters...)

* Section 5: Subcircuits (if needed)
.subckt subname pins...
...
.ends subname

.end
```

