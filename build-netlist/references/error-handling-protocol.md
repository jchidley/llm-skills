# Error Handling Protocol for Reverse Engineering

**Version:** v20251109
**Purpose:** Guide LLM behavior when encountering discrepancies, anomalies, and incomplete information during circuit reverse engineering.

---

## Core Principle

**Do NOT fix errors. Consult the human.**

Reverse engineering works with incomplete information. Errors and anomalies are expected—they often reveal important information about the actual circuit design, constraints, or undocumented features.

When you (the LLM) encounter something that doesn't fit expectations:
1. **Acknowledge it explicitly** - Name the discrepancy clearly
2. **Do not auto-correct** - Leave the anomaly in place
3. **Consult the human** - Ask what to do about it
4. **Preserve as data** - Treat "errors" as information, not problems to hide

---

## Two Critical Adjustments

### Adjustment 1: Don't Assume Previous Work Is Correct

**Problem:** The skill previously treated documented connections, netlist entries, and prior measurements as authoritative. This is wrong.

**Reality:** Errors are equally likely in past work as in current work. A connection documented three sessions ago may have been traced incorrectly. A component value entered into the netlist last week may be wrong.

**Correct Approach:**

When you encounter previous documentation or netlist entries, **do not assume they are correct**. Instead:

#### If Building on Previous Work
Ask clarifying questions to validate:
- "The netlist shows [X connection]. Can you confirm that's what you observe on the physical board?"
- "I see we documented [Y component value]. Have you been able to measure that since, or should we verify it?"
- "The previous trace says [Z is connected to W]. Does that match your current investigation?"

#### If Testing Reveals Discrepancies
Surface the contradiction explicitly:
- "The netlist says R14 = 107MΩ, but your color code reading shows 127Ω. These don't match. Which is correct?"
- "Documentation shows D9 as a diode, but you're describing it as a wire jumper. Which is it?"
- "The previous entry says Q11 emitter connects to [node], but your board shows it connects to [different node]. Which should we use?"

#### Build Confidence Incrementally
Only accept a connection as verified once:
1. You have physically observed it (or the user has)
2. You have confirmed it with the user
3. The user has indicated they're confident in that measurement

---

### Adjustment 2: Don't Fix "Errors"—Consult on What to Do

**Problem:** The skill (and LLM behavior generally) have a tendency to see something that doesn't match circuit theory expectations and auto-correct it without consultation.

**Examples of this mistake:**
- Seeing R18 = 712MΩ (extremely high value) and silently "fixing" it to 72MΩ because it seems more reasonable
- Finding Q12 configured as a low-side PNP switch (unusual) and assuming it "should" be something else
- Encountering D9 as a 0Ω wire jumper instead of a 1N4148 diode and changing it to make more sense
- Seeing a capacitor value that seems wrong and "correcting" it to standard values

**Why This Is Wrong:**
- **Anomalies are information.** An "incorrect" configuration often reveals something important about the real circuit design, cost constraints, board revisions, or specific performance requirements.
- **Your expectations may be wrong.** Just because something doesn't match textbook circuit theory doesn't mean the board is wrong—it might mean your understanding of the application is incomplete.
- **Incomplete information is normal.** You don't have visibility into the design intent, manufacturing history, or field failures that led to specific choices.
- **Fixing errors masks learning.** By correcting things without asking, you hide the very clues that would lead to deeper understanding of the circuit.

**Correct Approach:**

When you encounter something unexpected:

#### Step 1: Acknowledge It Explicitly
Name the anomaly clearly:
- "R18 is 712MΩ. That's an unusually high resistor value."
- "Q12 is configured as a low-side PNP switch (emitter grounded). This is uncommon."
- "D9 appears to be a 0Ω wire jumper instead of the 1N4148 diode shown on the schematic."

#### Step 2: Describe What You Observe
Provide context about why it seems unusual:
- "Typically, resistor dividers in this application use values in the kΩ range. 712MΩ is 1000× larger."
- "Most BJT low-side switches use NPN transistors with emitter grounded. PNP is typically high-side."
- "The PCB footprint suggests a 1N4148, but the physical component is a wire jumper."

#### Step 3: Consult the Human
Ask what to do:
- "Is this correct as-is, or should we investigate further?"
- "Do you want to measure this component directly to confirm the value?"
- "Should we leave this as a known anomaly and move on, or dig deeper?"
- "Does this configuration tell us something useful about how the circuit actually works?"

#### Step 4: Document and Move On
Depending on the user's response:
- **Continue as-is** - Document the anomaly in the netlist with a comment explaining why it's unusual
- **Investigate further** - Ask what additional measurements or observations would help resolve it
- **Defer decision** - Mark as "needs investigation" and move to other parts of the circuit
- **Treat as information** - Use the anomaly as a clue to understand other design choices

---

## When to Ask for Confirmation

### Always Ask When:
- A component value seems unreasonable (too high, too low, non-standard)
- A configuration is unusual (PNP in low-side, MOSFET in unexpected position, etc.)
- Previous documentation contradicts current observation
- Multiple possible interpretations exist for a connection
- The connection doesn't match the typical circuit topology for that application

### Do NOT Assume:
- Previous netlist entries are correct
- Documentation written in a prior session is accurate
- Component values match their color codes (measure them)
- A footprint's intended part matches what's actually soldered on
- Your expectations about "how this should work" match the actual design

### Do Prioritize Confirmation When:
- Testing has revealed new information
- The human has measured something directly
- A repair or modification has been made to the board
- You're building on previous work and something seems off

---

## Handling Different Types of Discrepancies

### Type 1: Value Discrepancies (Measurements Don't Match)

**Example:** "The netlist says R4 = 1.8Ω, but I just measured it at 1.2Ω."

**Response:**
- "The previous measurement was 1.8Ω, but your current measurement is 1.2Ω. Options:
  1. Take another measurement to confirm (test lead resistance ≈0.2-0.5Ω, so variance is normal for low-value resistors)
  2. Update the netlist to 1.2Ω if you're confident
  3. Use the color code if visible (often more reliable than ohmmeter for <10Ω resistors)

  Which approach would you prefer?"

### Type 2: Connection Discrepancies (Where Something Connects)

**Example:** "The netlist shows Q11 emitter at C1P, but your trace shows it at [different node]."

**Response:**
- "The previous trace had Q11 emitter connecting to C1P. You're showing a different connection. Can you trace it again to confirm? Is it possible the previous trace missed a jumper or pad?"

### Type 3: Configuration Discrepancies (How Something Is Used)

**Example:** "The netlist treats D9 as a diode, but you're describing it as a 0Ω wire jumper."

**Response:**
- "The PCB layout shows a 1N4148 footprint, but the actual component is a wire jumper (0Ω). This is interesting—it suggests either:
  1. The design was revised after layout (diode replaced with wire to bypass its voltage drop)
  2. The diode wasn't populated in this variant
  3. The wire was a field modification

  Do you know which it is? This might tell us something about design iterations or cost optimization."

### Type 4: Anomalies That Don't Make Sense

**Example:** "R18 = 712MΩ seems too high. Should this be 7.2MΩ or 72Ω?"

**Response:**
- "712MΩ is unusual. Before I change it, can you:
  1. Visually verify the color code: [list colors]
  2. Measure it directly if possible
  3. Check if this is in a feedback network where high impedance might actually be required

  Don't let me auto-correct this—unusual values often tell us something important. What does the board actually show?"

### Type 5: No-Connect (NC) vs. Floating Nodes

**Critical distinction:**
- **NC (no-connect) pad** = Deliberately unconnected by design (IC pin marked NC, unused footprint pad, etc.)
- **Floating node** = Accidentally undocumented terminal (ERROR)

**Example 1 - Legitimate NC:**
```
"U1 pin 7 is marked NC on the datasheet"
"R4 footprint is unpopulated (no component soldered)"
"Test point TP5 has only one pad soldered; other pad is intentionally open"
```

**Example 2 - Floating node (ERROR):**
```
"U1 pin 3 - ??? (don't know where it connects)"
"Q5 collector - I forgot to trace it"
"C12 right pad - (not sure what's there)"
```

**Naming Convention:**
- **Deliberate NC:** Use unique descriptive names: `NC_U1_PIN7`, `NC_R4_LEFT`, `NC_TESTPOINT_TP5`
- **Floating node (error):** Mark explicitly during fix-up phase: `FLOATING_[description]` then trace and resolve
- **Never use generic:** Avoid `NC`, `NC1`, `NC2` (loses the component context)

**When to Ask:**
- "Is [PIN/PAD] intentionally left unconnected, or should it be connected somewhere?"
- "The schematic shows this as NC. Does the physical board match (no solder, open pad)?"
- "I can't determine where [TERMINAL] connects. Is this a genuine NC pad?"

**Response Example:**
- If user confirms NC: "OK, I'll document this as `NC_U1_PIN7` to show it's deliberately unconnected."
- If user is unsure: "Let's trace the board to confirm. Can you look at what's on that pin/pad?"
- If it's floating (missing documentation): "This terminal is undocumented. I need to trace it or confirm it as NC before proceeding."

---

## When Waiting for More Information Is Valid

It's completely acceptable to **not fix something** and instead:

- **Document as-is** with a note: "Unusual configuration; revisit when more information available"
- **Mark for follow-up** in a separate list: "Questions to investigate when [X measurement is available]"
- **Continue with other parts** of the circuit while this ambiguity sits
- **Use the anomaly as a clue** - sometimes an odd configuration in one part of the circuit explains something else

Example documentation style:
```
.* ANOMALY: R18 = 712MΩ (unusually high for resistor divider)
.* Status: Verified by color code (Purple Brown Red Blue Brown ±1%)
.* Application: U2A regulator divider; high impedance may be intentional
.* Action: Revisit if LM1086 regulator output doesn't match expected voltage

R_Div_High C1P FB_Node 712Meg  ; Verified 2025-11-07, unusually high value
```

---

## LLM Behavior Change Required

This represents a fundamental shift in how the LLM should operate during reverse engineering:

**Old Behavior (Incorrect):**
- Assume previous work is correct
- Auto-fix anything that doesn't match expectations
- Hide anomalies rather than highlighting them
- Suppress uncertainty and present confident answers

**New Behavior (Correct):**
- Validate previous work against current observations
- Preserve "errors" and ask the human what to do
- Highlight anomalies as potentially meaningful information
- Explicitly acknowledge uncertainty and ask for clarification

**This applies to ALL reverse-engineering work**, not just SPICE netlist building.

---

## Implementation Checklist

When invoking `build-netlist` skill or doing reverse engineering:

- [ ] Am I assuming a previous measurement is correct without confirmation?
- [ ] Have I noticed something that doesn't match expectations and auto-corrected it?
- [ ] Did I hide an anomaly rather than surfacing it?
- [ ] Have I asked the human to confirm a discrepancy?
- [ ] Am I treating errors as information, not problems to hide?
- [ ] Have I documented anomalies with notes for future investigation?

---

## References

This protocol is based on:
- **Hardware reverse engineering best practices** - IPC standards emphasize documentation, not correction
- **Iterative design methodology** - Anomalies guide the next round of investigation
- **Incomplete information principle** - Engineering always works with partial knowledge
- **Error as information** - Manufacturing variability, design iterations, and field mods often leave "anomalies"

