# How Complexity Theory Relates to AI Agents

> **Convert expensive generation into cheap verification.**

## Overview

Building a correct thing is usually much harder than checking whether a thing is correct. Complexity theory gives that intuition a precise form, and engineering organisations can exploit it on purpose. They pay the cost of *generating* a good solution once, then turn it into a cheap, automated *check* that everyone can run many times.

This project is about that move: where the idea comes from, what it looks like in day-to-day engineering, and why it matters more now that AI agents generate code and infrastructure at volume.

**Start here:** [Convert Expensive Generation into Cheap Verification](convert-generation-into-verification.md)

---

## The Core Idea

1. **Generate once (expensive).** Someone with context and judgement works out a good solution: a secure module, an incident fix, an architectural decision, a reliability target.
2. **Encode it as a predicate (also done once).** Capture the solution as something a machine or a hurried human can check: a test, a policy rule, a template, a checklist, a threshold.
3. **Verify forever (cheap and repeated).** Every future change is checked against that predicate, with no need to rebuild the original reasoning.

The payoff comes from step 3 repeating. Writing good predicates is the high-leverage work.

---

## The Background, Stated Carefully

- **NP** is the class of decision problems where a "yes" answer comes with a certificate that can be checked in polynomial time. **P** is the class that can be *solved* in polynomial time. P ⊆ NP.
- **Whether P = NP is an open problem.** It's widely believed that P ≠ NP, which would mean some problems really are easier to check than to solve, but nobody has proved it.
- **Search spaces grow exponentially.** *n* independent yes/no choices give 2ⁿ combinations: about 10³ at *n* = 10, 10⁶ at *n* = 20 and 10³⁰ at *n* = 100. Checking a single candidate stays polynomial in the input size.
- **Most engineering tasks aren't formal NP problems.** Applying this to software design is an analogy. It explains why the asymmetry feels familiar, but it doesn't prove it applies.
- **Verification isn't automatically cheap.** Deciding non-trivial semantic properties of arbitrary programs is undecidable (Rice's theorem). What is cheap is checking a **narrow, explicit predicate**, such as "does this test pass?" or "does this plan expose a bucket publicly?"

---

## Practices That Make the Conversion

| Practice | Expensive generation it captures | Cheap check it enables |
|---|---|---|
| **Golden modules** | Secure, well-structured infrastructure design | "Are we on a supported version of the module?" plus `terraform test` / Terratest |
| **Policy-as-code** (OPA/Rego, Conftest, Sentinel, Checkov) | Security and compliance knowledge | Rules evaluated against every plan in CI |
| **ADRs + fitness functions** | Architectural reasoning and trade-offs | "Does this change conflict with an accepted decision?" plus automated architecture tests (ArchUnit, dependency-cruiser, import-linter) |
| **Runbooks** | Incident diagnosis and remediation | "Did step 4 bring the queue depth down?" |
| **SLOs and error budgets** | Product judgement about how reliable is reliable enough | `SLI ≥ target` over a window; burn-rate alerts |
| **Mutation testing** (PIT, Stryker, mutmut) | Confidence that tests mean something | Mutation score: do the tests catch injected faults? |

Each one is covered in detail, with examples and pitfalls, in the [essay](convert-generation-into-verification.md).

---

## Why It Matters for AI Agents

AI coding agents produce candidate solutions quickly and cheaply, which moves the bottleneck to verification.

- **Your checks are the agent's real specification.** An agent working in a generate-and-test loop is only as good as the tests, policies and gates inside that loop.
- **Weak checks get gamed.** Optimisers exploit gaps in their objective (specification gaming). With coding agents, that looks like special-casing test inputs, loosening assertions or deleting failing tests. Keep checks outside the agent's write scope where possible, and use mutation testing to measure whether they're strong.
- **Prefer deterministic checks over model judgement.** Compilers, type checkers, tests, policy rules and schema validators are sound gates. Using an AI model to judge another model's output is useful for triage, but it isn't a reliable verifier.

---

## Limits

- **Some work stays generative.** New product definitions, novel architectures and unfamiliar failure modes can't be reduced to a checklist in advance. The conversion applies to the *recurring* parts of the work.
- **Checks need upkeep.** Policies go stale, runbooks drift and SLOs need revisiting.
- **A green check isn't proof of correctness.** It shows that the specific properties you encoded hold, and nothing more.

---

## Repository Structure

```
How-Complexity-Theory-Relates-to-AI-Agents/
├── README.md                                   # This overview
├── convert-generation-into-verification.md     # Main essay (start here)
├── Expanded white paper.pdf                    # Earlier long-form draft (see note below)
├── Whitepaper(1).pdf                           # Earlier draft
├── paper_ait.docx                              # Earlier draft source
├── complexity_theory_ai_diagrams.html          # Exploratory Mermaid diagrams
├── complexity_accessible_diagrams (1).png      # Diagram export
└── .github/ISSUE_TEMPLATE/                     # Issue templates
```

> **Note on the earlier drafts:** The PDFs, the `.docx` and the diagrams are earlier, broader explorations. They use quantum-mechanics metaphors, and some of their complexity-theory statements are loose or incorrect. For example, they say verification "runs in P" while generation "lives in NP", and they claim LLMs reliably excel at verification. Treat the essay and this README as the current, corrected version of the argument.

---

## Further Reading

- Sanjeev Arora and Boaz Barak, *Computational Complexity: A Modern Approach*
- Michael Nygard, *Documenting Architecture Decisions* (2011)
- Neal Ford, Rebecca Parsons and Patrick Kua, *Building Evolutionary Architectures*
- Betsy Beyer et al., *Site Reliability Engineering* and *The Site Reliability Workbook*
- Open Policy Agent documentation: openpolicyagent.org

---

## Contributing

Corrections, worked examples from real teams, and new "generation → verification" conversions are all welcome. Please open an issue to suggest one.

## License

Released under **CC BY 4.0**. You may share and adapt the material for any purpose, provided you give appropriate credit.
