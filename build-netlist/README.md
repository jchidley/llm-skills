# Building SPICE Netlists from Circuit Descriptions

A comprehensive Claude Skill for systematically building SPICE netlists from circuit board descriptions through iterative tracing and verification.

## Skill Structure

This skill follows the Anthropic Agent Skills specification with progressive disclosure:

```
build-netlist/
├── SKILL.md                    # Main entrypoint (150 lines)
│   └── Metadata, quick start, essential rules, key reference links
├── references/                 # Detailed documentation (loaded as needed)
│   ├── pithy-rules.md         # 10 essential rules with explanations
│   ├── process-steps.md       # Detailed 6-step workflow
│   ├── naming-conventions.md  # Naming strategy and junction bloat prevention
│   ├── qspice-guide.md        # QSPICE simulation and testing
│   └── common-mistakes.md     # 12 common netlist errors with fixes
└── assets/                    # Supporting materials
    └── example-netlists.md    # Working examples from real tracing
```

## Key Features

- **Progressive Disclosure:** SKILL.md (150 lines) → Reference files loaded as needed
- **Comprehensive:** From basic concepts to QSPICE behavioral sources
- **Practical:** 12 common mistakes with fixes, 5 worked examples
- **Research-Backed:** Insights from Auto-SPICE, SPICEAssistant, Masala-CHAI
- **Standards-Aligned:** Follows Anthropic Agent Skills specification

## File Sizes

| File | Lines | Purpose |
|------|-------|---------|
| SKILL.md | 149 | Main entrypoint, quick start, rule summaries |
| pithy-rules.md | 113 | 10 essential rules with explanations |
| process-steps.md | 125 | 6-step workflow with detailed guidance |
| naming-conventions.md | 181 | Naming strategy, preventing junction bloat |
| qspice-guide.md | 235 | QSPICE execution, behavioral sources, validation |
| common-mistakes.md | 334 | Error patterns and prevention |
| example-netlists.md | 322 | 5 worked examples |
| **Total** | **1,459** | Complete skill package |

## Quick Start

1. Read SKILL.md for overview and essential rules
2. For detailed workflow: Load references/process-steps.md
3. For naming guidance: Load references/naming-conventions.md
4. For simulation: Load references/qspice-guide.md
5. For error prevention: Load references/common-mistakes.md
6. For examples: Load assets/example-netlists.md

## Core Principles

- **Explicit Over Implicit:** Never assume topology
- **Both Pads Always:** Document every component's two connections
- **Functional Names:** GATE_BIAS not NODE_A (2-4 components per junction)
- **Ask Before Confirming:** "What else is here?" prevents junction bloat
- **Physical References:** Left/right or top/bottom, used consistently
- **DC Paths Required:** Every node must connect to GND
- **Verify Before Simulating:** Read back netlist, confirm topology

## 10 Essential Rules

1. Explicit over implicit - never assume topology
2. Every component has two sides - document both
3. Vague names = wrong netlist - use functional names
4. Ask "What else is here?" - prevent junction bloat
5. Physical reference system - be consistent
6. Direction matters - series vs parallel vs divider
7. DC path to ground required - non-negotiable
8. Hierarchical organization scales - use sections
9. Names describe function - not location
10. Verify before finalizing - read it back

## QSPICE Integration

This skill is optimized for QSPICE (Qorvo's advanced SPICE engine):

- Behavioral sources for simplified netlists
- Direct resistor current access
- Running on WSL/Debian Windows
- Simulation validation workflow

## Standards Compliance

- SPICE3-compatible netlist syntax
- Follows Anthropic Agent Skills specification
- 3rd person description for skill discovery
- Imperative/infinitive writing style
- Progressive disclosure pattern

## When to Use

- Reverse-engineering circuit boards from physical descriptions
- Building netlists incrementally (no schematic available)
- Verifying netlist accuracy against board layout
- Documenting complex circuit interconnections
- Working with QSPICE to simulate and validate

---

*Last updated: 2025-11-09*
*Compatible with Claude Code and Claude Agent Skills system*
