# Build-Netlist Skill Adjustment Summary

**Date:** 2025-11-09
**Version:** v20251109
**Status:** ✅ Complete - Two critical behavioral adjustments implemented

---

## What Changed

The `build-netlist` skill has been adjusted to address two critical issues discovered during testing.

### Issue 1: Assuming Previous Work Is Correct

**Problem Found:**
When reverse-engineering a board, it's common to make mistakes. However, the skill treated previous documentation and netlist entries as authoritative truth. This is dangerous because errors in past sessions are just as likely as errors in the current session.

**Adjustment Made:**
The skill now validates all previous work against current observations:
- Ask clarifying questions: "The netlist shows [X]. What do you observe?"
- Don't accept documented connections without confirmation
- Build confidence incrementally—only mark something verified once physically confirmed
- Treat all entries (old and new) as potentially containing errors

**Implementation:**
- New reference file: `/references/error-handling-protocol.md`
- Added explicit guidance in main SKILL.md (section: "⚠️ CRITICAL: Error Handling Protocol")
- Adjustment 1 documented with examples and validation questions

---

### Issue 2: Auto-Correcting "Errors" Without Consulting User

**Problem Found:**
When the LLM encounters something that doesn't match circuit theory expectations, it tends to silently "fix" it. Examples:
- Seeing R18 = 712MΩ (unusual value) and "correcting" it to something "reasonable"
- Finding Q12 in an uncommon configuration and changing it to match textbook theory
- Seeing D9 as a wire jumper (not a diode) and "fixing" it

This is fundamentally wrong because **anomalies are information**. They reveal design intent, constraints, cost optimization, field modifications, or undocumented features.

**Adjustment Made:**
The skill now treats anomalies as data to be preserved and studied:
- Acknowledge anomalies explicitly ("R18 is 712MΩ, which is unusually high")
- Describe context ("Typically values are in the kΩ range; this is 1000× larger")
- Consult the user ("Should we investigate? Defer? Accept as-is?")
- Document with comments for future investigation
- **Never auto-correct without asking**

**Implementation:**
- New reference file: `/references/error-handling-protocol.md` (detailed guidance)
- Added comparison table showing wrong vs. correct behavior in main SKILL.md
- Adjustment 2 documented with decision tree and documentation templates
- Explicit statement: "Waiting for more information is always valid"

---

## New Files Created

### `/references/error-handling-protocol.md`

Comprehensive guide covering:

1. **Core Principle** - Do not fix errors; consult the human
2. **Adjustment 1 Details** - Validating previous work with examples
3. **Adjustment 2 Details** - Handling anomalies as information
4. **Five Types of Discrepancies** - How to handle value, connection, configuration, anomaly discrepancies
5. **When to Ask for Confirmation** - Checklist of scenarios
6. **LLM Behavior Change Required** - Old (incorrect) vs. New (correct) behavior
7. **Implementation Checklist** - Self-assessment questions for reverse engineering

**Key Sections:**
- Type 1: Value Discrepancies (Measurements Don't Match)
- Type 2: Connection Discrepancies (Where Something Connects)
- Type 3: Configuration Discrepancies (How Something Is Used)
- Type 4: Anomalies That Don't Make Sense

---

## Updated Files

### `/SKILL.md` (Main documentation)

Added prominent section: **⚠️ CRITICAL: Error Handling Protocol for Reverse Engineering**

Key additions:
- Side-by-side comparison of wrong vs. correct approaches
- Clear guidance on when to ask for confirmation
- Explicit statement on waiting for information
- Link to comprehensive protocol reference
- Updated Version History with MAJOR callout

---

## Practical Implications

### Before (Incorrect Behavior)
```
See R18 = 712MΩ
→ Thinks: "This doesn't seem right"
→ Silently "fixes" it to 72MΩ
→ Continues as if problem solved
→ User never learns about the anomaly
```

### After (Correct Behavior)
```
See R18 = 712MΩ
→ Acknowledges: "This is unusually high for a resistor divider"
→ Provides context: "Typically kΩ range; this is 1000× larger"
→ Asks user: "Is this correct? Should we investigate?"
→ Documents for future sessions
→ Anomaly preserved as information
```

---

## How to Use These Adjustments

When invoking the `build-netlist` skill or doing reverse engineering work:

1. **Review previous work with skepticism** - Don't assume it's correct
2. **Ask confirming questions** - "The netlist shows X. What do you observe?"
3. **Preserve anomalies** - Don't auto-correct unusual values or configurations
4. **Surface discrepancies** - Highlight contradictions explicitly
5. **Consult before deciding** - Let the user decide what to do about oddities
6. **Document everything** - Include comments explaining unusual decisions

---

## Version History Context

| Version | Date | Status | Key Changes |
|---------|------|--------|-------------|
| v20251107 | 2025-11-07 | Superseded | Core 6-step process, 10 essential rules |
| v20251109 | 2025-11-09 | Current | **Critical behavioral adjustments** |

The v20251109 update represents a fundamental shift in how the LLM should approach reverse engineering:
- **From:** Trust previous work, auto-correct anomalies
- **To:** Validate everything, preserve anomalies, consult user

---

## Validation Checklist

Use this checklist when working on reverse engineering:

- [ ] Have I confirmed previous documentation against current observations?
- [ ] Have I noticed something unusual and left it as-is?
- [ ] Have I asked the user about discrepancies?
- [ ] Have I documented anomalies with explanatory comments?
- [ ] Have I treated errors as information, not problems?
- [ ] Have I avoided auto-correcting without consultation?

---

## Questions?

Refer to `/references/error-handling-protocol.md` for comprehensive guidance on:
- How to validate previous work
- How to handle discrepancies
- When to ask for confirmation
- How to document anomalies
- Examples for each type of error

