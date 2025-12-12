# Gentle Engineering Notes

Long-form engineering notes on modern iOS: Swift, SwiftUI, concurrency, testing, and architecture.

---

## Authorship & Approach

The prose in these notes is largely produced through collaborative drafting with large language models.

I use LLMs to generate initial explanations, explore alternative formulations, and iteratively rewrite sections for clarity and structure. In many cases, the final text is the result of multiple guided rewrites rather than line-by-line manual authorship.

What *is* mine — and what I explicitly stand behind — are:
- the topics chosen,
- the architectural positions taken,
- the trade-offs emphasized,
- the examples included or excluded,
- and the practical guidance distilled from production experience.

I treat LLMs as a high-leverage writing tool, similar to pair programming or technical editing, while retaining full responsibility for technical accuracy, judgment, and intent.

These notes are intended as practical, experience-informed guides rather than academic papers or formal specifications.  
They evolve over time as my understanding deepens and as the platform changes.

---

## Directory Structure

```
├── 01_foundations/
│   ├── best_practices/      # Architecture & patterns
│   └── guides/              # How-to references
├── 02_architecture_and_design/
│   ├── architectural_guidance/  # Architectural patterns & approaches
│   ├── design_documents/        # System design specs
│   ├── feasability_studies/     # Exploratory research
│   └── research_concepts/       # Technical explorations
└── 03_testing_and_reliability/
    └── testing/                 # Testing strategies
```

---

## Articles

### 01 Foundations

#### Best Practices
- [Async Architecture](01_foundations/best_practices/async_architecture.md)
- [Data Modeling](01_foundations/best_practices/data_modeling.md)
- [State Management](01_foundations/best_practices/state_management.md)
- [Error Handling & Reliability](01_foundations/best_practices/error_handling_reliability.md)
- [Local Persistence & Caching](01_foundations/best_practices/local_persistence_caching.md)

#### Guides
- [Actors & Concurrency](01_foundations/guides/actors-concurrency-guide.md)
- [AsyncStream Basics](01_foundations/guides/asyncstream-basics-guide.md)
- [Codable & JSON](01_foundations/guides/codable-json-guide.md)
- [FileManager & File I/O](01_foundations/guides/filemanager-file-io-guide.md)
- [URLSession Networking](01_foundations/guides/urlsession-networking-guide.md)

### 02 Architecture & Design

#### Architectural Guidance
- [TCA-Inspired State Machine MVVM](02_architecture_and_design/architectural_guidance/tca_inspired_state_machine_mvvm.md)

#### Design Documents
- [StoreKit 2 Async Service](02_architecture_and_design/design_documents/storekit2_async_service_design.md)

#### Feasibility Studies
- [Voice-First Hybrid UI](02_architecture_and_design/feasability_studies/voice_ui_first_hybrid_ui.md)

#### Research Concepts
- [NEAR Token Internal Ledger](02_architecture_and_design/research_concepts/near_accounting_token_internal_ledger.md)

### 03 Testing & Reliability

#### Testing
- [Resilience Testing for iOS UI](03_testing_and_reliability/testing/ios-resilience-ui-testing.md)
- [Headless Integration Testing](03_testing_and_reliability/testing/headless_integration_testing_for_mobile_apps.md)
- [Modular UI Testing with Skeleton Apps](03_testing_and_reliability/testing/modular_ui_testing_with_skeleton_apps.md)

---

## 🦒 Gentle Giraffe Engineering

This repository is part of the broader **Gentle Giraffe Engineering** ecosystem:

- **GentleNetworking** — Swift networking abstractions  
- **SmartAsyncImage** — High-performance async image loading & caching  
- **Gentle-Engineering-Notes** — Long-form architecture & testing notes (this repo)

All projects emphasize:
- Production realism
- Testability
- Clear architectural boundaries
- Long-term maintainability

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.

---

## 📄 License

MIT License — free to use, reference, and adapt with attribution.

