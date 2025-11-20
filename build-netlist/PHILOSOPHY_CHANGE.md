# Philosophy Change: From "Two Sides" to "All Terminals"

## The Problem We Were Trying to Solve

The original Rule 2 was: **"Every component has two sides. Document both before moving on."**

This rule was trying to prevent **floating nodes**, which are the #1 SPICE simulation killer:
- Floating nodes cause convergence errors
- Floating nodes produce undefined voltages
- Floating nodes make simulation results meaningless

## Why "Two Sides" Was Too Narrow

The "two sides" formulation worked for **passive two-terminal components** (R, C, L, D) but:

1. **Doesn't apply to active components:**
   - BJTs have 3 terminals (collector, base, emitter) - not "two sides"
   - MOSFETs have 4 terminals (drain, gate, source, bulk) - not "two sides"
   - ICs have N pins - not "two sides"

2. **Hides the real problem:**
   - The real problem isn't "two sides" - it's **undocumented terminals**
   - A MOSFET with 3 documented terminals and 1 floating = broken netlist
   - A two-resistor divider with both sides documented + 5 other components = complex but valid

3. **Doesn't account for legitimate NC (no-connect):**
   - Some pins are intentionally unconnected (IC datasheets mark these as NC)
   - Some footprints are intentionally unpopulated (cost/performance tradeoff)
   - These aren't "missing documentation" - they're **deliberate design choices**

4. **Doesn't give clear guidance for edge cases:**
   - "What about test points with only one pad soldered?"
   - "What about IC pins marked as NC in the datasheet?"
   - "What about bulk/substrate pins on MOSFETs?"

## The Root Cause: Proxy vs. Principle

**Rule 2 was a PROXY** for the real principle, which is:

> **Every terminal of every component must have a documented state.**

Each state is one of:
1. **Connected to a signal node** (e.g., `Q1_COLL`)
2. **Deliberately NC** with unique name (e.g., `NC_U1_PIN7`)
3. **Tied to power/ground** (e.g., `VCC`, `GND`)

## The New Formulation

**Instead of "two sides":**

```
Document all terminals of every component. Every terminal must either be:
(1) connected to a named node,
(2) deliberately NC with unique name, or
(3) tied to power/ground explicitly.
```

This:
- ✅ Applies correctly to components with any number of terminals
- ✅ Captures the underlying problem (floating nodes break simulation)
- ✅ Provides clear guidance for edge cases (NC pads get unique names)
- ✅ Makes design intent visible (NC vs forgotten)

## The NC Naming Innovation

The original formulation had no way to distinguish:

```spice
R5 VCC ???                      ; Did we forget to document this?
                                ; Or is it intentionally unconnected?
```

The new NC convention makes this explicit:

```spice
R5 VCC NC_R5_PAD2              ; Intentionally unconnected (by design)
R5 VCC FLOATING_R5_PAD2         ; Accidentally undocumented (ERROR)
```

**Why unique naming?**

Generic NC names lose critical information:

```spice
❌ Bad:
R4 VCC NC 10k
R5 VCC NC 22k
* Which NC belongs to which resistor? Impossible to tell.

✅ Good:
R4 VCC NC_R4_PAD2 10k
R5 VCC NC_R5_PAD2 22k
* Immediately clear which pad is unconnected on which resistor
```

## Examples of the Shift

### Example 1: Passive Component (Still Works)

**Old rule:** "Document both sides of R5"

**New rule:** "Document both terminals of R5"

**Practical difference:** None for two-terminal components. Both formulations work. But the new one also covers multi-terminal components.

### Example 2: BJT (Didn't Work Before)

**Old rule:** "Doesn't apply - BJTs have three sides"

**New rule:** "Document all three terminals (collector, base, emitter)"

```spice
Q1 Q1_COLL BASE_BIAS GND Q_2N3906
    ↑      ↑           ↑
    all three terminals documented
```

### Example 3: IC Pin Marked NC in Datasheet (Ambiguous Before)

**Old rule:** "Is pin 7 NC, or did we forget to document it?"

**New rule:** "Explicitly show it as NC with unique name"

```spice
U1 IN FB NC_U1_PIN7 OUT GND ...
         ↑↑
         Shows pin 7 is deliberately NC (from datasheet)
```

### Example 4: Unpopulated Footprint (Ambiguous Before)

**Old rule:** "Is R34 missing, or is there a pad we didn't trace?"

**New rule:** "Show both pads, mark the empty one as NC"

```spice
R34 VCC NC_R34_PAD2 10k
         ↑↑
         Right pad is unconnected (footprint unpopulated)
```

## Key Insight: The Problem Isn't the Components

The problem is **information about the circuit**. When you reverse-engineer a board:

1. Every component exists
2. Every terminal has a state (connected, NC, or floating)
3. Your job is to **document the state explicitly**

The new formulation makes this crystal clear.

## Implementation Impact

**For circuit tracers:**
1. When tracing, document every terminal
2. If uncertain, mark as `FLOATING_[name]` temporarily
3. Before finalizing, resolve all FLOATING nodes
4. Intentionally NC pads get unique `NC_[COMPONENT]_[PAD]` names

**For SPICE simulation:**
1. No floating nodes in final netlist (simulation will converge)
2. NC nodes are documented (design intent is clear)
3. Every net is traceable (no mystery connections)

**For documentation:**
1. Netlist is self-documenting (no schematics needed)
2. NC pads are obvious (not hidden in generic names)
3. Future work understands the intent (not just "something is NC")

## Backwards Compatibility

These changes are **100% backwards compatible** with existing netlists:
- Two-terminal components work as before
- NC naming doesn't break anything
- The skill still applies to any component type

The changes simply make the framework:
- More precise about what "documented" means
- More explicit about intentional vs accidental NC
- More scalable to components with >2 terminals

## Conclusion

This isn't a rule change - it's **refactoring a proxy into a principle**.

**Old:** "Every component has two sides" (proxy, assumes all components are two-terminal)

**New:** "Every terminal must have a documented state" (principle, applies to all components)

The underlying problem (floating nodes break SPICE) remains the same. The solution (explicit terminal documentation) is clearer and more general.
