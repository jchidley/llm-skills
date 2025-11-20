# QSPICE Simulation & Testing Guide

After building a netlist, validate and simulate it using QSPICE (Qorvo's advanced SPICE engine).

## Running QSPICE on WSL/Debian Windows

Execute QSPICE simulations from WSL (Windows Subsystem for Linux):

```bash
'/mnt/c/Program Files/QSPICE/QSPICE64.exe' your_circuit.cir
```

**Real example from Life Fitness board:**
```bash
'/mnt/c/Program Files/QSPICE/QSPICE64.exe' life_fitness_board_fixed.cir
```

## Quick Netlist Checklist Before Running

Verify these 10 items before attempting simulation:

- [ ] All components have unique names (R1, R2, not R1, R1)
- [ ] All node names spelled consistently (GND vs gnd = error)
- [ ] Power supplies defined (V_sources present)
- [ ] Ground node present (GND or 0)
- [ ] All models referenced are defined
- [ ] `.end` statement present at end
- [ ] Comments use `*` or `;`
- [ ] All values have units (10k, 47n, 100m)
- [ ] Analysis directive present (`.tran`, `.ac`, `.dc`, `.op`)
- [ ] No floating nodes (every node has DC path to GND)

## Common QSPICE Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Node floating" | No DC path to GND | Add R or V source to GND, or high-value pullup |
| "Unknown model" | Referenced model not defined | Add `.model` directive or `.include` file |
| "Syntax error" | Malformed component line | Check format against syntax reference |
| Convergence fails | Too-large time steps | Reduce `.tran` max step parameter |
| "Too many iterations" | Difficult to solve circuit | Try `.option method=gear` for stability |

## QSPICE-Specific Features

### Behavioral Sources (B Sources)

QSPICE's unique strength: arbitrary mathematical expressions in netlist.

```spice
BV_FB FB GND V=V(VOUT)/10              * Voltage division
BI_SENSE Sense GND I=I(R_Shunt)        * Access resistor current
BR_Dynamic N1 N2 R=1k + V(Ctrl)*100    * Voltage-dependent resistor
```

**Supported in expressions:**
- Node voltages: `V(node)`, `V(node1,node2)` for differential
- Component currents: `I(Rname)`, `Ib(Q1)`, `Ic(Q1)` for transistor pins
- Math: `+`, `-`, `*`, `/`, `^`, `abs()`, `sqrt()`, `sin()`, `cos()`, `exp()`, `ln()`, `log10()`, `min()`, `max()`
- Time: `time` variable for time-dependent expressions

**Examples:**
```spice
* Feedback voltage divider
BV_FB FB GND V=V(VOUT)/10

* Controlled current source (VCCS)
BI_LOAD OUT GND I=V(IN)/1k

* Multiplier for nonlinear gain
BV_GAIN OUT GND V=V(IN)*V(CTRL)/10

* Voltage comparator with hysteresis
BV_CMP OUT GND V=IF(V(IN)>1, 10, 0)

* Dynamic resistor (voltage-dependent)
BR_NONLIN N1 N2 R=1k*(1 + 0.5*V(CTRL))

* Time-varying signal
BV_SWEEP IN GND V=5*sin(2*pi*1k*time)
```

### Behavioral Source Performance Optimization

Not all behavioral sources are equal. Performance and convergence differ significantly.

**Voltage-Controlled Current Source (VCCS) vs VCVS:**

Prefer G-source (VCCS) with shunt resistor over E-source (VCVS) for faster convergence:

**Slow (VCVS—Voltage-Controlled Voltage Source):**
```spice
E_Feedback FB GND VOUT GND 0.1    * Forces voltage node—harder to solve
```

**Faster (VCCS + Resistor):**
```spice
G_Feedback FB GND VOUT GND 0.1    * Current-based—converges better
R_FB_SHUNT FB GND 10k              * 0.1 * 10V / 10k = 100µA bias
```

**Why:** Current sources are mathematically easier for solvers than voltage-enforcing sources. Use G-sources whenever possible.

**Breaking Complex Expressions:**

Large, complex behavioral expressions slow down solvers. Break them into simpler components:

**Slow (one complex expression):**
```spice
BV_OUT OUT GND V=(V(IN1)*V(IN2)*(V(CTRL)+0.5))/(1+abs(V(FB)))
```

**Faster (simple components):**
```spice
BV_INT1 INT1 GND V=V(IN1)*V(IN2)          * Intermediate node 1
BV_INT2 INT2 GND V=V(INT1)*(V(CTRL)+0.5) * Intermediate node 2
BV_OUT OUT GND V=V(INT2)/(1+abs(V(FB)))   * Final output
```

### Handling Discontinuities in Behavioral Sources

The `if()` statement creates discontinuities that force the solver to reduce time steps dramatically, slowing simulation.

**Problem (abrupt step):**
```spice
BV_STEP OUT GND V=IF(V(CTRL)>5, 10, 0)    * Step function causes issues
```

**Solution (smooth with capacitor):**
```spice
BV_STEP OUT GND V=IF(V(CTRL)>5, 10, 0)
C_SMOOTH OUT GND 10p                       * Small cap softens transition
```

**Why:** The capacitor doesn't change circuit behavior significantly but converts the abrupt voltage change into a smoother exponential transition, allowing larger time steps and faster convergence.

### Integration Method Selection

Choose between Gear (stable) and Trapezoid (fast, default):

```spice
.option method=trap    * Trapezoid integration (default, faster)
.option method=gear    * Gear integration (more stable)
```

Start with trapezoid; switch to gear if convergence issues occur.

### Tolerance Analysis (Parameter Variations)

Validate circuit robustness to component manufacturing variations using Monte Carlo or worst-case analysis.

**Monte Carlo Analysis (Random Variations):**

Most realistic for production validation. Randomly vary component values within tolerance bands:

```spice
* Define nominal and tolerance values
.param R1_nom 10k
.param R1_tol 0.05              * 5% tolerance

* Use mc() function for Monte Carlo
.param R1_val {mc(R1_nom, R1_tol)}
.param C1_val {mc(100n, 0.10)}  * 100nF ± 10%

* Use parameters in netlist
R1 VCC BIAS {R1_val}
C1 BIAS GND {C1_val}

* Run 100 Monte Carlo iterations
.tran 0 1m 0 1u
.step param mc 1 100 1
```

**Why Monte Carlo:** Tests real-world distribution of parts. QSPICE randomly samples from distribution for each iteration.

**Worst-Case Analysis (Extreme Values):**

Find circuit behavior at extreme component values (all high or all low):

```spice
* Test all resistances high, capacitances low
.step param R1_val list {10k*1.05} {10k*0.95} param C1_val list {100n*0.90} {100n*1.10}

* Or explicitly test extreme conditions
.alter case=1_RHI_CLO
  R1 10.5k   * R high
  C1 90n     * C low
.endalter

.alter case=2_RLO_CHI
  R1 9.5k    * R low
  C1 110n    * C high
.endalter
```

**Industry Standard:** 5% tolerance metrics more reliable than exact parameter matching for validation (per SPICEAssistant research).

### .bode Analysis (Switching Circuit Frequency Response)

For switching converters and circuits with complex feedback, use `.bode` for AC perturbation analysis with built-in FFT:

**When to use .bode:**
- Switching power supplies with PWM feedback
- Complex closed-loop circuits where linear AC analysis fails
- Verifying control loop stability
- Measuring frequency response with large-signal switching

**Basic .bode Setup:**

```spice
* Perturb duty cycle around operating point
.bode dec 100 1 1Meg V(OUT)     * Frequency sweep 1Hz-1MHz, 100 pts/decade
```

**Advanced .bode Options (Oct 2024+):**

```spice
* Debug mode: preserve time-domain waveforms for analysis
.bode DEBUG V(OUT)

* Square wave perturbation (instead of sine)
.bode SQUARE 1 V(OUT)
```

**Common Pitfall - Perturbating Source Placement:**

❌ **WRONG - Perturbating source directly to ground:**
```spice
V_PERTURB IN 0 AC 1              * Directly to ground
```

✅ **CORRECT - Use series connection:**
```spice
V_SERIES IN IN_DUMMY DC 0 AC 1   * Series 0V source with AC perturbation
V_DUMMY IN_DUMMY 0 AC 0          * Reference voltage source
```

**Why:** Direct ground connection may bypass circuit paths. Series connection ensures perturbation propagates through the feedback network properly.

**Split Sweeps for Wide Frequency Ranges:**

For very wide ranges (1Hz to 1MHz), split into multiple .bode runs to maintain resolution:

```spice
* Decade 1: 1Hz to 1kHz (100 points)
.bode dec 100 1 1k V(OUT)
* Decade 2: 1kHz to 1MHz (100 points)
.bode dec 100 1k 1Meg V(OUT)
```

This prevents aliasing and provides better resolution than single wide sweep.

### Direct Resistor Current Access

Unlike standard SPICE, QSPICE allows behavioral sources to reference resistor currents:

```spice
* Access current through resistor R_SENSE directly
BI_MONITOR FB GND I=I(R_SENSE)
```

This simplifies current-sense modeling without needing explicit current sources.

### C++ and Verilog Integration

QSPICE supports embedding compiled code for high-performance custom components (advanced feature).

```spice
* Symbol configuration uses Ø (.dll) device type
* Port names must match Verilog module declarations
X_VERILOG_MODULE out1 out2 in1 in2 VERILOG_BLOCK
```

### Enhanced Device Modeling

Native support for advanced semiconductors (GaN, SiC):
- Gate leakage current modeling
- Subthreshold conduction
- Linear region accuracy improvements

## Example QSPICE Netlist

```spice
Life Fitness Controller Board - Power Stage
* Circuit description and revision date

* Power sources
V_ALT C1P GND DC 12
V_LOGIC PWM GND PULSE(0 10 0 10n 10n 5u 10u)

* Main components
R4 C1P Q7_SOURCE 0.51
M_Q7 Q7_DRAIN Q7_GATE Q7_SOURCE Q7_SOURCE NMOS_2N03L05

* Analysis
.tran 0 1m 0 1u

* Model definitions
.model NMOS_2N03L05 NMOS (...)

.end
```

## Typical QSPICE Testing Workflow

1. **Build and validate netlist**
   - Create `.cir` file with all components and connections
   - Verify junction names are consistent
   - Confirm all node references exist

2. **Add analysis commands**
   - `.tran` for transient response (time-domain)
   - `.ac` for frequency response (AC analysis)
   - `.dc` for DC operating point
   - `.op` for operating point only

3. **Run simulation**
   ```bash
   '/mnt/c/Program Files/QSPICE/QSPICE64.exe' your_circuit.cir
   ```

4. **Check for errors**
   - Look for "netlist error" messages
   - Verify node connectivity (floating nodes)
   - Confirm all referenced models are defined

5. **Analyze results**
   - QSPICE generates `.raw` output files
   - Use QSPICE's plotting tool to view waveforms
   - Extract numerical data for verification

## QSPICE Netlist Structure

QSPICE netlists follow standard SPICE with these requirements:

1. **Header/Title** - First line is a comment describing circuit
2. **Component Definitions** - Standard SPICE syntax (R, C, D, Q, M, V, I, B)
3. **Simulation Commands** - `.tran`, `.ac`, `.dc` for analysis types
4. **Model Definitions** - `.model` directives for custom component models
5. **Subcircuits** - `.subckt` for reusable circuit blocks (optional)
6. **Include Directives** - `.include` or `.lib` to load external model files
7. **End Statement** - `.end` terminates the netlist (required)

## Best Practices for QSPICE Netlists

1. **Use Behavioral Sources for Control Logic**
   - Replace complex analog circuits with B sources
   - Reduces simulation time and improves convergence
   - Example: Replace feedback network with `BV_FB FB GND V=V(OUT)/10`

2. **Leverage Direct Resistor Current Access**
   - Use `I(Rname)` in behavioral sources for current monitoring
   - Simplifies current-sense modeling

3. **Employ Correct Integration Method**
   - Start with default trapezoid (faster)
   - Switch to gear if convergence issues occur

4. **Use Step Rejection Sparingly**
   - Add `tripdv` and `tripdt` only if convergence problems occur
   - Overly strict values can slow simulation dramatically

```spice
* Step rejection example (use sparingly)
B_name node1 node2 V=<expression> tripdv=1 tripdt=1m
* tripdv: max voltage change per tripdv time window (volts)
* tripdt: time window for voltage change detection (seconds)
```

5. **Test Verilog/C++ Code Separately**
   - Compile and verify custom modules before integrating
   - Use simple test vectors first

## Analysis Directives

Standard QSPICE analysis commands:

```spice
.tran 0 1 0 1u                * Transient: 0-1s time, 1µs max step
.ac dec 100 1 1Meg            * AC: decade sweep, 100 pts/decade, 1Hz-1MHz
.dc V1 0 5 0.1                * DC: sweep V1 from 0-5V, 0.1V steps
.op                           * Operating point only

.four 1k 10 V(OUT)            * Fourier analysis at 1kHz fundamental
```

## LLM Research Insights

Recent research (SPICEAssistant) shows:
- **38% accuracy improvement** with simulation feedback loop
- Build → test → iterate is more effective than aiming for perfect first draft
- Simulation errors guide netlist corrections

**Implication:** Don't aim for perfect netlist on first pass. Build, simulate, fix, repeat.

