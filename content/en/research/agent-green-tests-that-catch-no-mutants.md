---
title: "Every Sixth 'Proven' Acceptance Criterion by a Coding Agent Relies on a Test That Catches No Breakages"
date: '2026-10-28'
description: "We took 300 acceptance criteria that an autonomous agent marked as proven across 12 of its products and broke the code under each with mutations. In 16.3% [12.3; 20.3] of cases, the test remained green despite any breakage, confirmed by a test that catches the mutant. Two LLM judges, examining the test text, do not find such cases: κ between them is 0.57."
tags: ["AI","LLM","Coding Agents","Testing","Mutation Testing","Research"]
authors: ['ICNC Lab']
language: 'en'
---

An autonomous agent closes an acceptance criterion by writing code, then writing a test with the criterion's ID in its name; if the test is green, a checkmark is applied. We took 300 such checkmarks and, in each case, broke the code that the test was supposed to verify. In 69 cases, the test remained green, no matter what we broke.

Next, we needed to determine how many of these 69 were genuine and how many were artifacts of the method. We couldn't find people for labeling, and replacing humans with models yielded a second, more unpleasant result than the first.

## Corpus

Twelve products that our agent built using a single template (Bun, Elysia, PostgreSQL). Each product's specification is broken down into use cases, each with Acceptance Criteria (AC) at a specific verification level: unit, integration, or browser. In total, there are 3,143 ACs, with 2,368 marked as proven. There are 2,057 commits, of which 1,497 are agent-generated.

The agent works with slices: a slice is a vertical segment of a product with its own code directory and its own ACs. The 'proven' mark is set by a runner that reads a JUnit report and finds a green test with the criterion's ID in it. The checkmark cannot be faked manually. However, the runner only verifies the 'ID → green run' connection; it doesn't know if the test actually checks anything.

## K1: Mutations by Slice Code

From the proven ACs, we took 300 random ones with product quotas. For each, we generated mutants in the slice's code: deletion of `!`, replacement of comparisons, substitution of `return`, removal of calls, logic, arithmetic, `true/false`. We ran the associated test against each mutant. A mutant is killed if the test turned red.

We define a 'vacuously green' AC as one where the associated test killed no mutants. The threshold of interest was set in advance: the topic is worth an article if at least 10% of such cases are found.

A total of 2,817 mutants were generated, with 781 killed. The average mutation score per AC is 0.28. Vacuously green ACs accounted for 69 out of 300, which is 23.0% [18.7; 27.7].

For 22 of the 69, the associated test did not execute any lines of the slice's code: the implementation was in another module (e.g., an AC for `identity/users` slice was proven by a test in the `auth/` module), and mutants were missed. For these, we moved the mutations to lines of product code that the test *does* execute. For 20, the test killed at least one mutant; for 2, no mutation locations were found, and they were removed from the denominator. With this adjustment, the number of vacuously green ACs remaining is 69 − 22 = 47 out of 298, which is 15.8% [11.7; 19.8]. I devised this adjustment after observing the data, so it is presented as a sensitivity analysis.

The spread across products is large: 4.8% for one, 27.6% and 31.8% for two others. Unit tests are vacuously green more often than integration tests (17.7% vs. 14.6% with the adjustment).

## Where the Method Can Lie

An undetected mutant does not always imply a weak test. A mutant can be equivalent: `x >= 0` instead of `x > 0` where zero never occurs, the program doesn't change, and no test will catch it. In this case, 'vacuously green' means there was nothing to break.

The plan included a blind manual review of 10% by two people. However, no one was available for labeling, so K1b replaced humans with two checks, both with a predefined threshold before execution.

**Proof of Nonequivalence.** A model from a different family (`gpt-6.1-sol`, the agent's code was written by a Claude model) receives the AC text and slice code and writes its own test. If this test is green on the original code and red on at least one mutant, the mutant is definitely nonequivalent, and the original test is definitely vacuous. Threshold: confirm ≥70% of the 69 cases.

**LLM Judges instead of Humans.** Two judges from different families (`gpt-6-luna` and `deepseek-v4.1-flash`) read the AC and the body of the associated test and answer whether the test verifies the criterion. The sample consists of 30 ACs, with mutation verdicts hidden. Threshold: κ ≥ 0.6.

## A Test Cannot Be Written Blindly from AC Text

The preregistered part failed: 9 out of 69 were confirmed, 13.0% [7.0; 23.0].

This does not prove mutant equivalence. For 26 ACs, the writer's test was already red on the original code, both attempts. The model doesn't know fixtures: which columns are NOT NULL, which seeds are in the database, in what order to create related records. Furthermore, for 34 ACs, a valid test killed no mutants, but this test was also written blindly.

Then I gave the writer what is legitimate for the question of equivalence: the diff of each mutant and the entire associated test. Any valid test that distinguishes a mutant from the original code proves nonequivalence, regardless of how it was written. I added this option (K1b-S) to the README before its execution, but after observing intermediate data, so it is also presented as a sensitivity analysis.

| | AC | share |
|---|---|---|
| confirmed nonequivalent mutant | 49 out of 69 | 71.0% [59.4; 80.4] |
| valid targeted test killed no mutants | 4 | candidates for equivalent |
| no valid test found | 16 | unknown |

Four ACs where a valid targeted test failed to kill the mutant, and 16 ACs with no answer, could turn out to be equivalent-meaning, in the worst case, 20 out of 69. There are 49 confirmed vacuously green ACs. If we count only these as vacuous, the refined estimate is **16.3% [12.3; 20.3]** of 300 proven ACs. With the adjustment for tests outside the slice, it's **11.1% [7.7; 14.4]**. The central estimate remains above 10%, but the lower bound of the interval does not.

## LLM Judge Does Not See Vacuous Tests

The judges agreed with each other at κ = 0.567, with an exact match in 24 out of 30 cases. The threshold of 0.6 was not met.

With the mutation verdicts, the judges agreed in 83% and 90% of cases, but κ with it was -0.09 and 0.00. The agreement is based on the baseline frequency: in the sample, 27 out of 30 tests killed at least one mutant, and the judges almost always considered them as verifying. All three vacuously green ACs in the sample were deemed verifying by both judges. The test appears correct: it calls the necessary route, checks the status, names the criterion. However, from the text alone, it is not evident that it will not fail under any breakage.

Therefore, 'let a second model look at the agent's tests' is not suitable as an acceptance control: mutations catch what a judge by text alone misses.

## What It Cost

All of K1 ran on CPU, 1.7 hours on the longest product, with products processed in parallel. K1b cost $12.51: blind writer $6.51 (3.49 million input tokens, pre-run estimate was $3-6), mutant variant $5.98, judges $0.02.

## Limitations

- 47 ACs were excluded from the sample: 19 with fewer than three mutation locations, 18 with tests requiring an external service (OIDC), 10 with runs that were red in our environment due to manual DB state. This biases the sample towards tests that run in a clean environment.
- Proof of nonequivalence was obtained with a targeted test aware of the mutant, not blindly, and after intermediate data. 16 ACs remained without an answer.
- There is no blind human review. LLM judges do not replace it, and this result is negative in itself.
- A vacuously green test does not mean the function is broken. The test simply doesn't guard it: the next change might break it, and the checkmark will remain.
- The corpus is singular: 12 products from a single agent based on one template.

The agent's 'proven' checkmark is applied honestly by the runner upon a green test run. However, for approximately every sixth criterion, this checkmark is backed by a test that remained green despite every code breakage we introduced.