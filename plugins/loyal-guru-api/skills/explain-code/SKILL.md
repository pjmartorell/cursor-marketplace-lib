---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

# Explain Code

## Instructions

When explaining code, follow this structure:

### 1. Draw a diagram
Use ASCII art to show flow, structure, or relationships. Examples:

**Flow diagram:**
```
Request ──▶ Controller ──▶ Service ──▶ Database
                                  ◀──  Response
```

**Hierarchy:**
```
  BaseClass
  ├── ChildA
  │   ├── method_one
  │   └── method_two
  └── ChildB
      └── method_three
```

**Sequence:**
```
Client        Server        DB
  │──request──▶│              │
  │            │──query──────▶│
  │            │◀──results────│
  │◀─response──│              │
```

### 2. Walk through the code
Explain step-by-step what happens at runtime. Use numbered steps tied to specific lines or blocks.

### 3. Highlight a gotcha
Call out a common mistake, misconception, or subtle behavior that could trip someone up.

### 4. Ask if wants an analogy
Compare the code to something from everyday life to build intuition before diving into details.

## Style

- Keep explanations conversational.
- For complex concepts, layer multiple analogies.
- Adapt depth to the complexity of the code — simple code gets a brief explanation, complex code gets a thorough one.
