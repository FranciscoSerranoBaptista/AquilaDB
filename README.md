# AquilaDB

**The business question:** does our leadership development change how leaders lead, and can we show it without exposing a single person?

Most leadership development measures what is easy: attendance, satisfaction, a score at the end of a programme. It rarely measures whether behaviour changed. To measure change, three things must be in one place: the records, the relationships between them over time, and a way to ask questions of the whole population without seeing any one person. AquilaDB is that place.

This repository is the public face of the project: what it is for, screenshots and public issues. The source code is private.

## What a leadership function gets

- **Evidence, not attendance.** The records of development sit in one graph over time, so a programme can be read for change in behaviour, not only for satisfaction.
- **A population view.** Questions go to the whole leader population, by level, function or programme, never to one person.
- **Privacy a works council can read.** Identity fields are encrypted and can never be queried. No result comes back for a group smaller than five people. Records are de-identified before any AI step.
- **Questions agreed before the answers exist.** A study is written down, approved by the hash of its plan, and run once. Nobody fishes in the data until something looks good.
- **People decide, the model does not.** A language model can describe a result. Every number and status comes from a validation gate.

## How it works

AquilaDB is a multi-model graph database written in Rust. Documents, a property graph, full-text search and vector search live in one engine behind one API. It speaks a query language with broad ArangoDB AQL parity, with a public [tree-sitter grammar](https://github.com/FranciscoSerranoBaptista/tree-sitter-aql). Sensitive fields are encrypted at the value level, in tiers.

## The stack

| Part | The business question it answers |
|---|---|
| **AquilaDB** | Where do the records of development live, safely, over time? |
| **[KestrelAnalytics](https://github.com/FranciscoSerranoBaptista/KestrelAnalytics)** | What patterns hold across the population, and are they stable enough to act on? |
| **[PeacockUI](https://github.com/FranciscoSerranoBaptista/PeacockUI)** | Can a leadership team see those patterns and decide what to do next? |

## Status

`v0.1.0`, pre-release, single node. In use for my own practice research since May 2026.

## Screenshots

<!-- add images to docs/screenshots/ -->

## Issues

Use the issues here for questions, ideas and collaboration requests. The working backlog is private.

© 2026 Francisco Serrano Baptista. All rights reserved.
