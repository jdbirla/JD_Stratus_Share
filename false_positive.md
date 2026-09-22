# False Positive Detection

Yes. I finalized the architecture based on our discussion and created a **GitHub-ready design package**.

### What I included

```text
false-positive-detection-design/
├── README.md
├── GITHUB_COMMIT_GUIDE.md
└── diagrams/
    ├── architecture.mmd
    ├── sequence.mmd
    ├── duplicate-detection.mmd
    └── context-extraction.mmd
```

The design specifically reflects the approach we agreed on:

* ~1M alerts/day
* Maximum ~50 policies
* Channel-independent classification while retaining channel metadata
* Exact duplicate detection
* Template/near-duplicate detection
* **No embeddings/vector DB in the initial architecture**
* Alert-context extraction
* Policy-based token-aware batching
* LLM classification
* `TRUE_POSITIVE / FALSE_POSITIVE / UNCERTAIN`
* Human review for uncertain cases
* Auditability and metrics
* Future evolution based on actual production measurements

### Main architecture

```text
Communication Sources
 Outlook / Webex / Bloomberg / Other
                 │
                 ▼
       Existing Surveillance
       Policies + Rules + Keywords
                 │
                 ▼
        Alert Normalization
                 │
                 ▼
          Group by Policy
                 │
                 ▼
       Exact Duplicate Check
                 │
                 ▼
    Template / Near-Duplicate Check
                 │
                 ▼
       Representative Alerts
                 │
                 ▼
       Context Extraction
                 │
                 ▼
       Token-Aware Batching
                 │
                 ▼
        Policy-Aware LLM
                 │
                 ▼
     TP / FP / UNCERTAIN
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
       FP       TP    Uncertain
        │        │        │
        │        │        ▼
        │        │   Human Review
        └────────┴────────┘
                 │
                 ▼
          Results / Audit DB
```

### Download the package

[Download the GitHub-ready design package](sandbox:/mnt/data/false-positive-detection-design)

You can extract it and commit the contents directly to your repository. The `.mmd` files are **Mermaid diagrams**, which GitHub can render natively.

The `README.md` is also written as the initial architecture/design document, so you can use it directly as your project documentation.

**Next, I would recommend turning this into the actual technical design**: database tables + Python package structure + duplicate-detection algorithms + context-extraction implementation + LLM batch/prompt schema. That would give you a solid HLD → LLD path rather than just an architecture diagram.
