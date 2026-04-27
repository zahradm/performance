# Foundations of Software Testing (Senior Engineer Perspective)

Senior engineers don't treat testing as "QA's job." Testing is how a team **manages risk** and **protects delivery speed**.

## 1) Verification vs Validation

- **Verification**: "did we build it right?" (code review, type checks, linters)
- **Validation**: "did we build the right thing?" (user acceptance, product testing)

**Senior note:** Most outages are *spec gaps* or *assumption gaps*, not syntax errors.

## 2) A Test Strategy Is a Portfolio

```
Many:  unit tests (fast feedback, cheap)
Some:  service/API tests (contracts, integration)
Few:   end-to-end UI tests (critical journeys only)
Plus:  non-functional (performance, security, resilience)
```

## 3) Risk-Based Testing

You cannot test everything equally. Prioritize by:
- Business criticality (money, auth)
- Change frequency
- Complexity
- Blast radius

## 4) Test Reliability: Flaky Tests Are Technical Debt

Common causes:
- Time-based waits (use polls with deadlines)
- Shared mutable test data
- Reliance on external systems
- Non-deterministic concurrency

**Senior practice:**
- Deterministic assertions
- Isolated test data per run
- Hermetic tests where possible
- Quarantine + root-cause fixes for flaky tests

## 5) Interview Answers

**"How do you decide what to test?"**

"Risk-based. I protect critical journeys with stable tests, keep fast feedback at unit/service level, and rely on observability in production."

**"What's worse: missing tests or flaky tests?"**

"Flaky tests can be worse because they destroy trust in CI. I treat flakiness as a production issue."
