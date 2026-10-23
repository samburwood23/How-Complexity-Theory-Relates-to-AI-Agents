# Convert Expensive Generation into Cheap Verification

**Author:** Sam Burwood
**Tags:** platform-engineering, devops, testing, ai-agents, architecture
**Reading time:** ~12 minutes

---

## TL;DR

> Building a correct thing is usually much harder than checking whether a thing is correct. Strong engineering organisations exploit that gap on purpose. They pay the expensive cost of *generating* a good solution once, then turn it into a cheap, automated *check* that everyone else can run thousands of times. Golden modules, policy-as-code, Architecture Decision Records, runbooks, SLOs and mutation testing are all versions of the same move. AI coding agents make this matter more, not less: an agent's output can only be as good as the checks you put around it.

---

## The Idea in One Paragraph

Think about a sudoku. Solving one can take twenty minutes. Checking a finished grid takes about thirty seconds: every row, column and box contains 1–9 once. Most engineering work has the same shape. Designing a Terraform module that is secure, cost-bounded and easy to change takes days. Checking that a given plan doesn't open port 22 to the internet takes milliseconds. The lesson isn't "verification is easy, so relax." It's this: **wherever you can, invest generation effort in building verifiers, because a verifier can be reused indefinitely and a one-off solution can't.**

---

## A Short, Accurate Word on Complexity Theory

The intuition comes from computer science, so it's worth stating precisely, without stretching it.

- **NP** is the class of decision problems where a "yes" answer comes with a *certificate* (a proposed solution) that can be checked in polynomial time. Sudoku, SAT and graph colouring are examples.
- **P** is the class of problems that can be *solved* in polynomial time. Every problem in P is also in NP.
- **Whether P = NP is unproven.** Most researchers believe P ≠ NP, which would mean some problems are fundamentally easier to check than to solve. Nobody has proved it.
- **Search spaces grow exponentially.** With *n* independent yes/no design choices there are 2ⁿ combinations: about a thousand at *n* = 10, about a million at *n* = 20 and about 10³⁰ at *n* = 100. Checking one candidate costs roughly the same however large *n* gets. Enumerating them all doesn't.

Two caveats keep us honest:

1. **Most engineering tasks aren't formal NP problems.** "Design a good CI pipeline" has no crisp yes/no specification. The complexity-theory framing is an *analogy* that explains why the asymmetry feels so familiar. It isn't a proof that it applies.
2. **Verification isn't automatically cheap.** By Rice's theorem, deciding any non-trivial semantic property of an arbitrary program is undecidable. "Is this code correct?" has no general cheap checker. What *is* cheap is checking a **narrow, explicit predicate**: "does this test pass?", "does this plan create a public S3 bucket?", "is p99 latency under 300 ms?"

That second caveat is the whole game. **Cheap verification doesn't come for free. You build it, by turning vague quality goals into narrow predicates.** That turning is itself a generation task, and it's usually the most valuable one your senior engineers do.

---

## The Pattern

Every example below follows the same three steps:

1. **Generate once (expensive).** Someone with context and judgement works out a good solution: a secure module, an incident fix, an architectural decision, a reliability target.
2. **Encode it as a predicate (still expensive, done once).** Capture the solution in a form a machine or a hurried human can check: a test, a policy rule, a template, a checklist, a threshold.
3. **Verify forever (cheap, repeated).** Every future change gets checked against the predicate automatically or in minutes, with no need to regenerate the original reasoning.

The return on investment comes from step 3 repeating. A policy rule that took a day to write and runs on 5,000 pull requests a year costs very little per check.

---

## 1. Golden Modules

**The generation problem:** Writing infrastructure-as-code from scratch means getting dozens of decisions right every time: encryption defaults, IAM boundaries, tagging, logging, network exposure, lifecycle rules. Each team that does it independently pays that cost again and makes slightly different mistakes.

**The conversion:** A platform team writes a small set of **golden modules**. These are versioned, opinionated Terraform (or Pulumi, or CDK) modules for the common building blocks, such as "an S3 bucket", "a Postgres instance" or "a service on our cluster". Secure defaults are baked in and only the parameters that really vary are exposed.

**What gets verified, and how:**

- `terraform validate` and linters such as `tflint` check syntax and provider-specific mistakes.
- Module tests check the module behaves as intended. Terraform has had a native `terraform test` framework since v1.6, and Terratest (Go) covers deploy-and-assert integration tests.
- Semantic versioning in a private module registry lets consumers see whether an upgrade is safe.
- Consumers no longer answer "is this bucket configured securely?" They answer "are we on a supported version of the golden module?", which is a far cheaper question.

**Why it works:** The expensive reasoning about what "secure by default" means happened once, inside the module. Everyone downstream inherits it and only has to check that they're using it.

**Watch out for:** Golden modules that expose every underlying argument "for flexibility" turn back into raw resources with extra steps. The value lies in the decisions the module *removes*.

---

## 2. Policy-as-Code

**The generation problem:** A reviewer reading a 400-line infrastructure diff and asking "is this compliant with our security standards?" is doing slow, error-prone work, and it scales linearly with headcount.

**The conversion:** Write the standards as executable rules that run in CI against the *planned* change, before anything is applied.

- **Open Policy Agent (OPA)** is a CNCF-graduated, general-purpose policy engine with rules written in Rego. **Conftest** runs Rego policies against structured config files, including the JSON output of `terraform show -json plan.out`.
- **HashiCorp Sentinel** is the policy framework built into HCP Terraform and Terraform Enterprise.
- Scanners such as **Checkov** and **Trivy** ship with large libraries of pre-written misconfiguration checks.

A Rego rule looks like this:

```rego
package terraform.s3

deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.after.block_public_acls == false
  msg := sprintf("%s must block public ACLs", [rc.address])
}
```

**What gets verified:** Every plan, on every pull request, against every rule, in seconds.

**Why it works:** Someone generated the knowledge that public ACLs are a risk, and in which contexts. The rule captures it as a predicate. The reviewer's job shrinks from re-deriving security policy to handling the exceptions the policy flags.

**Watch out for:** Policy only checks what it encodes. A clean policy run means "none of the known-bad patterns are present". It doesn't mean "secure". Keep human review for the things you haven't written down yet, and turn recurring review comments into new rules.

---

## 3. Architecture Decision Records (ADRs)

**The generation problem:** Architectural decisions such as "why are we event-driven here?" or "why Postgres and not DynamoDB?" take weeks of exploration. Six months later the reasoning has been forgotten and the same debate starts again from scratch.

**The conversion:** An ADR is a short document, popularised by Michael Nygard in 2011, that records **Context, Decision, Status and Consequences** for one significant choice. ADRs live in the repo next to the code and are never edited after acceptance, only superseded by later ADRs.

**What gets verified:** When someone proposes a change, the question shifts from "what should our architecture be?" (generation) to "does this proposal conflict with an accepted ADR, and if so, has the context changed enough to supersede it?" (verification). Reviewers can answer that in minutes.

**Going further: fitness functions.** Some ADR consequences can be made machine-checkable. *Building Evolutionary Architectures* (Ford, Parsons and Kua) calls these **architectural fitness functions**: automated tests of architectural properties. Examples:

- ArchUnit (Java), `dependency-cruiser` (JS/TS) or `import-linter` (Python) rules asserting that "the `billing` package must not import from `web`".
- A CI check that no service calls another service's database directly.

That moves ADRs from "human-checkable" to "machine-checkable", which is the full conversion.

**Watch out for:** ADRs written after the fact to justify a decision already made. The value comes from recording the *alternatives considered* and the *context*, because that's what lets a future reader verify whether the decision still holds.

---

## 4. Runbooks

**The generation problem:** At 3 a.m., during an incident, "how do we fix this?" is the most expensive question you can ask. Every minute of open-ended diagnosis is customer impact, and tired people under pressure tend to fix the first plausible symptom they see.

**The conversion:** After each incident, write down the diagnosis and remediation as a runbook: an ordered set of checks and actions tied to a specific alert.

That turns

> "What is wrong and how do we fix it?" (generation)

into

> "Is the queue depth above 10k? Yes → did step 4 (scale consumers) bring it down within five minutes?" (verification)

**What gets verified:** Each step has an observable expected outcome, so the responder checks progress rather than inventing a plan.

**Going further:** Link every paging alert to its runbook. Treat "this alert has no runbook" as a defect. Automate the steps that are always the same, because an automated runbook step is just a remediation script with a check after it.

**Watch out for:** Runbooks go stale silently. Review them in post-incident reviews ("did the runbook help, and where was it wrong?") and exercise them in game days so the verification steps still match reality.

---

## 5. SLOs as Verification Predicates

**The generation problem:** "Is the service reliable enough?" is subjective, and arguing about it is endless. Product, engineering and leadership each have a different answer.

**The conversion:** A **Service Level Objective** turns that debate into a number. Following the Google SRE books:

- An **SLI** (indicator) is a measured ratio, for example `good requests / valid requests`, where "good" means status < 500 and latency < 300 ms.
- An **SLO** is the target for that SLI over a window, for example 99.9% over 30 days.
- The **error budget** is `1 − SLO`: here 0.1% of requests, or about 43 minutes of full downtime in 30 days.

Choosing the SLO is generation. It needs product judgement, user research, historical data and trade-off discussions. Once chosen, checking it is a single comparison:

```text
SLI_30d >= 0.999  →  within objective
```

**What gets verified:**

- **Burn-rate alerts** check how fast the budget is being consumed. The *SRE Workbook* recommends multi-window alerts. For example, a burn rate of 14.4× over one hour means 2% of a 30-day budget gone in that hour, which is worth paging someone.
- **Release decisions** become a check too. If the error budget is exhausted, the agreed policy (for example, freezing feature launches and prioritising reliability work) applies without a fresh negotiation.

**Why it works:** Reliability stops being a recurring argument and becomes a predicate anyone can evaluate from a dashboard.

**Watch out for:** SLOs on the wrong SLI. A latency SLO measured at the load balancer can be green while users time out in a client-side retry loop. Verification is only as meaningful as the predicate you chose.

---

## 6. Mutation Testing: Verifying the Verifier

Everything above depends on one assumption: **the checks actually catch problems.** A test suite with 95% line coverage can still assert nothing useful. Coverage tells you which code *ran*, not whether a bug in it would be *detected*.

**Mutation testing** checks the checker:

1. A tool makes small, deliberate changes (mutants) to your code, such as flipping `<` to `<=`, replacing `+` with `-`, or deleting a function call.
2. It runs your test suite against each mutant.
3. If a test fails, the mutant is **killed**: your tests detected the change. If every test still passes, the mutant **survived**: you have a bug your tests can't see.
4. The **mutation score** is the share of non-equivalent mutants that were killed.

Mature tools include **PIT** (Java), **Stryker** (JavaScript/TypeScript, C#, Scala) and **mutmut** (Python).

**Why it belongs in this post:** Mutation testing is the quality-control step for your verification investment. Golden modules, fitness functions and CI gates all turn generation into verification, and mutation testing tells you whether that verification is real.

**Practical tips:**

- Running mutation testing over a whole codebase is expensive. Run it on changed files per pull request, or nightly on critical modules.
- Some mutants are **equivalent**: they don't change behaviour, so no test can kill them. Detecting these is undecidable in general, so expect some manual triage and don't chase 100%.
- Treat surviving mutants in critical paths (auth, billing, data deletion) as test-gap tickets.

---

## Why This Matters More with AI Agents

AI coding agents produce candidate solutions quickly and in volume. That doesn't remove the gap between generating and verifying. It **moves the bottleneck to verification**.

A useful mental model for an agent working on code is a **generate-and-test loop**: propose a change, run the checks, read the failures, revise. The quality of what comes out of that loop is bounded by the quality of the checks inside it. Three consequences follow.

**1. Your verifiers are now your agent's specification.**
If the only check is "tests pass", the agent will optimise for tests passing. Golden modules, policy-as-code, fitness functions and SLO gates give an agent the same guardrails they give a human, and they do it mechanically, without anyone having to remember to ask.

**2. Weak verifiers get gamed.**
Optimisers exploit gaps in their objective. In ML this is called *specification gaming* or *reward hacking*. With coding agents, it looks like special-casing a test input, loosening an assertion, or deleting a failing test. Mitigations:

- Keep the verifier outside the agent's write scope where you can, for example with protected paths or CODEOWNERS on test and policy directories, and required status checks on the branch.
- Use mutation testing to measure whether the tests the agent relies on can actually detect regressions.
- Review diffs to test files with extra suspicion.

**3. Prefer deterministic checks over model judgement.**
Using an LLM to review another LLM's output ("LLM-as-judge") can help with triage, but it isn't a sound verifier. It can be inconsistent, sensitive to phrasing, and wrong in the same ways the generator is. Where a deterministic predicate exists (a compiler, a type checker, a test, a policy rule, a schema validator, a `terraform plan` diff), use it as the gate and keep model review advisory.

The organisations that get the most out of agents won't be the ones with the cleverest prompts. They'll be the ones that already did the work of turning their engineering judgement into checks.

---

## How to Start on Monday

You don't need a platform team to begin. Pick the cheapest conversion that pays off fastest:

| If you keep repeating… | Convert it into… | First step |
|---|---|---|
| The same review comment on infra PRs | A policy-as-code rule | Write one Conftest/Checkov rule for your most common comment |
| Copy-pasted Terraform for the same resource | A golden module | Extract the most-copied resource into a versioned module with tests |
| The same architecture debate | An ADR (+ a fitness function) | Write an ADR for the last decision you re-litigated |
| The same 3 a.m. diagnosis | A runbook | Link your noisiest alert to a written runbook |
| "Is it reliable enough?" arguments | An SLO with an error budget | Define one SLI and a 30-day target for your most critical endpoint |
| Doubts about whether your tests catch anything | Mutation testing | Run a mutation tool on one critical module and read the survivors |

A simple rule of thumb: **every time you notice yourself working something out for the second time, ask what predicate would have let you just check it.**

---

## Limits Worth Admitting

- **Some work stays generative.** Defining what "good" means for a new product, choosing a new architecture, or diagnosing a novel failure mode can't be reduced to a checklist ahead of time. Conversion applies to the *recurring* parts of the work.
- **Verifiers cost money to maintain.** Policies go stale, runbooks drift, SLOs need revisiting as products change. Budget for upkeep, or the checks will quietly stop meaning anything.
- **A green check is not proof of correctness.** It's proof that the specific properties you encoded hold. Be explicit about what each check covers so nobody over-trusts it.

---

## Takeaways

1. **Checking is usually cheaper than building, but only for narrow, explicit predicates.** Writing those predicates is the high-leverage work.
2. **Pay generation costs once, then verify many times.** Golden modules, policy-as-code, ADRs, runbooks and SLOs are all ways to amortise expert judgement.
3. **Verify your verifiers.** Mutation testing tells you whether your checks would actually catch a bug.
4. **AI agents raise the stakes.** Agents generate cheaply, so your checks become the real specification. Make them deterministic, hard to tamper with, and strong.

---

## Discussion

- Which review comment do you leave most often, and could it be a policy rule instead?
- Which of your alerts would a new on-call engineer struggle with tonight?
- If you pointed a coding agent at your repo tomorrow, what would stop it from "passing" by weakening a test?

---

## Further Reading

- Michael Nygard, *Documenting Architecture Decisions* (2011)
- Neal Ford, Rebecca Parsons, Patrick Kua, *Building Evolutionary Architectures* (O'Reilly)
- Betsy Beyer et al., *Site Reliability Engineering* and *The Site Reliability Workbook* (Google / O'Reilly), especially the chapters on SLOs and alerting on SLOs
- Open Policy Agent documentation: openpolicyagent.org
- Terraform documentation: `terraform test` and module registries
- Sanjeev Arora and Boaz Barak, *Computational Complexity: A Modern Approach*, for the formal background on P, NP and verification
