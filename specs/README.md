---
role: authored
---
# Classifier specifications

What `arbol-classifier` does: it has a text judged by Jev for an [Arbol Trunk](../../arbol/GLOSSARY.md#arbol-trunk). The architecture of Arbol 2 says where the component stands ([ARC-0](../../arbol/architecture/00-components-and-boundaries.md), "Small model calls"); the specification here says what it takes, gives and sends.

Conventions (shape of a specification, requirement identifiers, origin, checks, status, decisions) are defined once in the [Driver Contract's README](../../arbol/driver-contract/README.md#conventions). Identifiers here start with `CLS`.

| Spec | Title | Status |
|---|---|---|
| [CLS-0](00-the-classifier.md) | The classifier | implemented |

The specification rests on a reading of Jev's public documentation on 13 October 2026, recorded in CLS-0 under "What Jev's documentation says". The first real question to Jev, on 14 October 2026, proved the API reference right on the one point where the documentation contradicts itself.
