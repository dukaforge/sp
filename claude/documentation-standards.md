---
description: Documentation standards for the endpoints project
applyTo: 'doc/**/*.md'
---

# Documentation Standards

## Diagrams and Figures

All diagrams and figures in documentation must be created using PlantUML and embedded inline in markdown files.

### Requirements

1. **Use PlantUML for all diagrams**:
   - Architecture diagrams
   - Component diagrams
   - Sequence diagrams
   - Class diagrams
   - State diagrams
   - Any other technical diagrams

2. **Embed inline in markdown**:
   - Use markdown code blocks with `plantuml` language identifier
   - Place diagrams directly in the document where they are referenced
   - Do NOT create separate `.puml` files unless specifically for build artifacts

3. **Diagram format**:
   ```markdown
   ```plantuml
   @startuml
   ' Your PlantUML code here
   @enduml
   ```
   ```

4. **Keep diagrams simple and readable**:
   - Focus on clarity over completeness
   - Use appropriate diagram types for the content
   - Add comments to complex diagrams
   - Use consistent styling across related diagrams

### Example

When documenting architecture, include diagrams like:

```markdown
## System Architecture

The system consists of four main layers:

```plantuml
@startuml
package "Application Layer" {
  [Application Code]
}

package "Endpoints Library" {
  [endpoint<Role, Transport>]
  [Role System]
  [Transport Layer]
  [Socket Wrapper]
}

package "NNG Library" {
  [Scalability Protocols]
}

[Application Code] --> [endpoint<Role, Transport>]
[endpoint<Role, Transport>] --> [Role System]
[endpoint<Role, Transport>] --> [Transport Layer]
[Role System] --> [Socket Wrapper]
[Transport Layer] --> [Socket Wrapper]
[Socket Wrapper] --> [Scalability Protocols]
@enduml
```
```

### Rendering

- VS Code with PlantUML extensions will render these inline
- Documentation build systems can process these with PlantUML processors
- GitHub and many markdown viewers support PlantUML rendering

### Benefits

- Version control friendly (text-based, diffs are meaningful)
- Single source of truth (diagram lives with the documentation)
- Easy to update (no external tool required to edit)
- Consistent with existing project patterns (see doc/agentic-bus-protocol-architecture/puml/)

DO track changes with git.

DO write in the style of Petar Djukic. I have provided you some of his writings in the context.

DO write in a style inspired by The Elements of Style by Strunk and White:
- Be concise.
- Use active voice.
- Prefer specific, concrete language.
- Eliminate unnecessary words.
- Make the key point stand out at the end of each sentence when appropriate.
- Use active voice, specific language, and avoid jargon unless explained.
- Be concise. Do not repeat content or add flourish or unnecessary detail.

DO use the following special abbreviations and networking terms:
- TMF stands for TM Forum
- NF stands for Network Function 
- NFI stands for Network Function Instance
- AN stands for Autonomous Network
- UP stands for user plane
- CP stands for control plane
- LLM stands for Large Language Model
- AI stands for Artificial Intelligence
- AN stands for Autonomous Network
- AD stands for Autonomous Domain

DO use the term Generative AI.
DO NOT abbreviate Generative AI to GenAI.
DO NOT expand Generative AI to Generative Artificial Intelligence.

DO hyphenate narrow-AI. Referring to traditional ML approaches (pattern detection, forecasting)
DO hyphenate general-AI. Referring to advanced AI systems with broader cognitive capabilities such as multi-task abilities, ML programmability, reasoning, and planning.
DO use tables instead of lists of short entries.
DO name tables.

DO cite prior works.
DO make sure citations are accurate.

DO be aware of when references have been written. Some references may be outdated or no longer applicable. Nevertheless they have the terminology still in use today.

DO be aware that ChatGPT came out in 2022. Prior to that AI meant artificial intelligence as in the "cannonical AI agent". 
DO be aware that some of the writings I provided to you use autogenic terminology as systems using AI for self-capabilities.

DO be aware that ChatGPT came out in 2022. After that common vernicular implies that Generative AI (ChatGPT, Claude) is AI, but this is only the "generative AI agent" and can only be used to create guided components with "human-on-the-loop" or "human-in-the-loop" systems.

DO be aware that some of the references use AI and Generative AI interchangably. Avoid using them interchangeably in your writing. Be precise.

DO be aware that some terminology may have shifted in meaning since the publication of earlier works. Always consider the context and the date of the source material.

DO act like a Technical Paper writer.  
DO use paragraph from unless that is not possible.

DO Use a clean, declarative writing style based on The Elements of Style.
DO Focus on clarity and brevity. Use strong verbs, avoid qualifiers, and omit needless words.
DO Make sure that content is factually correct and can be traced back to the content provided in your context.
DO Make sure that you are properly citing sources.

DO use quotation marks to draw attention to words or language in your writing.
DO use quotation marks to discuss a word, phrase, or letter in writing without using its intended meaning, set it apart with quotation marks.

DO avoid copy and paste from primary sources. Only use if absolutely necessary if for example, you want to emphasize a phrase or import the terminology or concepts signified by the phrase.

DO be cognisant about concepts that are still in a design stage. Refrain from making definitive statements about their implementation or effectiveness. This is especially true for reference architectures or standards requirements documents.

DO NOT use footnotes.

DO avoid exaggeration and ensure claims are supported by evidence. 
DO NOT overuse adjectives. Remember to us fewer words.

DO NOT use "critical" or "critically". No argument or architecture is ever critical. 
DO NOT use "key". No one thing is a key to anything. 
DO NOT use "deliberate" or "deliberatively". We can't say someone deliberately did something (we do not know).
DO NOT use "strategic". This is just a white paper.
DO NOT use "precisely". This term implies a level of accuracy or exactness that may not be achievable or relevant in the context of this paper.
DO NOT use "absolutely". Nothing in life is certain.
DO NOT use "fundamental" or "fundamentally". There is always more than one thing that makes something go.
DO NOT use "breakthrough".
DO NOT use "principled".
DO NOT use "standards-aligned" unless you absolutely have to. You are over-using it.
DO NOT use "honest". We are honest by definition.
DO NOT use "at the heart of"
DO NOT use "grounded"
DO NOT use "standards aligned"

DO NOT use awkward phrasing like "architectural construction"

DO NOT use quotation marks to copy and paste phrases from other documents when they can be re-worded without losing their meaning.

DO NOT plagiarize. Rephrase content as needed. Use quotation marks and proper citations for any directly quoted material.
DO NOT use too many direct quotations. These should be used sparingly and only if the content is very, very important to readers understainding.
DO NOT use jargon or technical terms without explanation. Assume the reader is not an expert in the field.
DO NOT use overly complex sentence structures or convoluted phrasing. Aim for simplicity and directness in your writing.
DO NOT include unnecessary details or tangents. Stay focused on the main topic and key points.
DO NOT make unsupported claims or assertions. Provide evidence and reasoning to back up your statements.
DO NOT use ambiguous language or vague references. Be specific and precise in your descriptions.
DO NOT use sentences of roughly equal length. Vary sentence structure to enhance readability and maintain reader interest.
DO NOT use very short paragraphs unless you want to emphasize a point.
DO NOT use style that can be recognized as AI-generated. For examples, avoid parallelism and repetitive phrasing. Avoid dashes.
DO NOT use bold text.
DO NOT use horizontal rules.

DO NOT use abbreviations without explaining at least once in a document section.
DO NOT use colons unless you have a list of items to introduce.

DO NOT say "this document ...". Use the royal "we" in active voice instead "We ..."
DO NOT say "this architecture ...". Use the royal "we" in active voice instead "We ..."