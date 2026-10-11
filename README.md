# mermaid-pl

Plan for a small local LLM that turns a short Polish description into a Mermaid diagram.
Design only so far, no code yet.

## Goal

- Input: a short request in Polish, e.g. "flowchart obsługi reklamacji z dwiema decyzjami"
- Output: one `mermaid` code block, nothing else
- Labels in Polish, node ids in ASCII
- Diagram types: flowchart, sequence, class, state, ER, gantt, mindmap
- The diagram must parse in both Mermaid 10.9 and 12.x, so no new-only syntax and no styling
- Model around 8B parameters (around 4B if 8B does not fit), trained on one 24 GB GPU

Target: at least 95% valid diagrams on the first try and 99% after the repair loop,
better than the untuned base model and not worse than larger local models.

## Why it is not trivial

- There is no Polish text-to-Mermaid dataset, so the data has to be created.
- Fine-tuning alone barely improves syntax validity. Feeding the parser error back to the model helps much more.
- If the reward is only "does it parse", the model learns to write short, poor diagrams.

## Data

1. Take existing Mermaid code from open datasets and keep only diagrams that pass validation.
2. A teacher model translates the labels to Polish without touching the structure.
3. The teacher writes Polish requests for each diagram in different styles (full spec, brief, casual).
4. Add synthetic examples for diagram types and domains that are poorly covered.
5. Add repair examples: request, broken diagram, parser error, fixed diagram.

About 25k training examples, plus separate sets for DPO, validation and testing.

## Validator

A small Node service that runs `mermaid.parse()` for both Mermaid versions and returns the error message
and the diagram structure (nodes and edges). A render check catches diagrams that parse but come out blank.

## Training

1. SFT with LoRA.
2. DPO: the tuned model generates several candidates per request. They are scored by validity,
   coverage of the elements named in the request, and a judge model. Best and worst form a training pair.

At inference there is a repair loop: generate, validate, on error ask the model to fix it (up to 3 rounds).

## Evaluation

- Own Polish test set (about 600 requests) plus Polish translations of MermaidSeqBench and VisPlotBench
- Metrics: valid on first try, valid after repair, node and edge F1 against the reference, LLM judge score
- Compared systems: base model, after SFT, after SFT+DPO, and larger local reference models

## Steps

1. Validator and data download
2. Pick the teacher and the base model on a small pilot
3. Build the full dataset
4. SFT
5. DPO
6. Report and a small CLI tool
