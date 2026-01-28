# Command: Bootstrap Project Documentation

I'm starting a new project and need you to help me create VISION.md and ARCHITECTURE.md files.

Ask me questions to understand:
1. What problem I'm trying to solve
2. What the solution will do
3. What success looks like
4. What the major components are
5. How those components fit together
6. Key design decisions and why

Based on my answers, create:

VISION.md with this structure:
- ## The Problem (what pain point we're solving)
- ## What This Does (the solution in 2-3 paragraphs)
- ## What Success Looks Like (concrete success criteria)
- ## What This Is NOT (explicit non-goals)

ARCHITECTURE.md with this structure:
- ## High-Level Overview (component diagram in text/ASCII)
- ## Component Responsibilities (each major component and what it does)
- ## Data Flow (how information moves through the system with concrete example)
- ## Key Design Decisions (why we chose this approach, not alternatives)
- ## Technology Choices (languages, frameworks, protocols, tools)
- ## Project Structure (directory layout)
- ## What's Next (implementation order or next steps)

**Important: Diagrams and Figures**
- Use inline PlantUML diagrams embedded directly in the markdown files
- Wrap PlantUML source in ```plantuml code blocks
- DO NOT use image references like ![diagram](path/to/image.png)
- Include diagrams for: system context, component relationships, data flows, state machines, sequence diagrams
- Use simple, clean PlantUML syntax with !theme plain and white background
- Diagrams should be self-explanatory with clear titles and notes

Keep both documents concise and focused. VISION should be 1-2 pages, ARCHITECTURE should be 2-4 pages.

Start by asking me questions to understand the project.