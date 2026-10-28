---
title: "One in Six 'Proven' Acceptance Criteria from a Coding Agent Rely on a Test That Catches No Breakages"
date: '2026-10-28'
description: "We analyzed 300 acceptance criteria marked as 'proven' by an autonomous agent across 12 of its products, deliberately introducing code breakages through mutations. For 16.3% [12.3; 20.3] of these, the associated test remained green despite any introduced breakage, a finding further validated by a mutant-catching test. Two LLM judges, evaluating solely on test text, failed to identify these cases, with a κ (Cohen's kappa) of 0.57 between them."
tags: ["AI","LLM","Coding Agents","Testing","Mutation Testing","Research"]
authors: ['ICNC Lab']
language: 'en'
---

An autonomous agent closes an acceptance criterion by: writing code, writing a test with the criterion's ID in its name, the test is green, and a checkbox is marked. We took 300 such checkboxes and, in each case, broke the code that the test was supposed to verify. In 69 cases, the test remained green, no matter what we broke.

Next, we needed to determine how many of these 69 were genuine, and how many were an artifact of the method. No people were found for annotation, and replacing people with models yielded a second result, more unpleasant than the first.

## Corpus

Twelve products that our agent assembled on a single template (Bun, Elysia, PostgreSQL). Each product's specification is divided into use cases, with each having acceptance criteria (AC) at a verification level: unit, integration, or browser. In total, there are 3,143 ACs, with 2,368 marked as proven. There are 2,057 commits, of which 1,497 are agent-generated.

The agent works with slices: a slice is a vertical piece of a product with its own code directory and its own ACs. The 'proven' mark is set by a runner that reads the JUnit report and finds a green test with the criterion's ID in it. The checkbox cannot be faked manually. However, the runner only verifies the 'ID → green run' connection; it does not know if the test actually verifies anything.

## K1: Mutations on Slice Code

From the proven ACs, we took 300 random ones with quotas per product. For each, we generated mutants in the slice's code: removal of `!`, replacement of comparisons, substitution of `return`, removal of calls, logic, arithmetic, `true/false`. The associated test was run against each mutant. A mutant is killed if the test turned red.

We define an 'empty green' AC as one whose associated test did not kill any mutants. The threshold of interest was set beforehand: the topic is worth an article if at least 10% of such cases are found.

2,817 mutants were generated, 781 were killed. The average mutation score per AC was 0.28. There were 69 empty greens out of 300, which is 23.0% [18.7; 27.7].

For 22 out of 69, the associated test did not execute any lines of the slice's code: the implementation lay in another module (e.g., an AC of the `identity/users` slice was proven by a test from the `auth/` module), and the mutants were missed. If mutations were moved to the lines executed by the test, all 20 such ACs, where there was something to mutate, passed the tests. With this adjustment, the result was 15.8% [11.7; 19.8]. I conceived this adjustment after seeing the data, so it is presented as a sensitivity analysis.

The spread across products is large: 4.8% for one, 27.6% and 31.8% for two others. Unit tests are more often empty than integration tests (17.7% vs 14.6% with adjustment).

## Where the Method Can Lie

An undetected mutant does not always mean a weak test. A mutant can be equivalent: `x >= 0` instead of `x > 0` where zero never occurs, the program does not change, and no test will catch it. In such cases, an 'empty green' means there was nothing to break.

The plan included a blind manual review of 10% by two people. No one was available for annotation, so K1b replaced human reviewers with two checks, both with pre-defined thresholds.

**Proof of Non-Equivalence.** A model from a different family (`gpt-6.1-sol`, the agent's code was written by a Claude model) receives the AC text and slice code and writes its own test. If the test is green on the original code and red on at least one mutant, the mutant is definitely non-equivalent, and the original test is definitely empty. Threshold: confirm ≥70% of the 69.

**LLM Judges Instead of Humans.** Two judges from different families (`gpt-6-luna` and `deepseek-v4.1-flash`) read the AC and the body of the associated test and answer whether the test verifies the criterion. Sample: 30 ACs, mutation verdict hidden. Threshold: κ ≥ 0.6.

## A Test Cannot Be Written Blindly From AC Text

The pre-registered part failed: 9 out of 69 were confirmed, 13.0% [7.0; 23.0].

This does not prove mutant equivalence. For 26 ACs, the writer's test was already red on the original code, both attempts. The model doesn't know fixtures: which columns are NOT NULL, which seeds are in the database, in what order to create related records. For another 34, a valid test killed no mutants, but it too was written blindly.

Then I gave the writer what is legitimate for the question of equivalence: the diff of each mutant and the associated test in its entirety. Any valid test that distinguishes a mutant from the original proves non-equivalence, regardless of how it was written. I added this variant (K1b-S) to the README before its launch, but after intermediate data, so it is also treated as a sensitivity analysis.

| | AC | share |
|---|---|---|
| confirmed non-equivalent mutant | 49 of 69 | 71.0% [59.4; 80.4] |
| valid targeted test killed no mutants | 4 | candidates for equivalent |
| no valid test | 16 | unknown |

Equivalence explains no more than 4 out of 69 empty greens. If we consider only the confirmed 49 as empty, the refined estimate is **16.3% [12.3; 20.3]** of 300 proven ACs. With the adjustment for tests outside the slice, it's **11.1% [7.7; 14.4]**. The central estimate remains above 10%, but the lower bound of the interval does not.

## LLM Judge Does Not See Empty Test

The judges agreed with each other with κ = 0.567, an exact match in 24 out of 30 cases. The threshold of 0.6 was not met.

With the mutation verdict, the judges coincided in 83% and 90% of cases, but their κ values with it were −0.09 and 0.00. The coincidence is maintained by the baseline frequency: in the sample, 27 out of 30 tests killed at least one mutant, and the judges recognized almost everything as validating. All three empty greens from the sample were recognized by both judges as validating ACs. The test looks correct: it calls the necessary route, checks the status, names the criterion. What is not visible from the text is that it won't fail under any breakage.

Therefore, 'let a second model look at the agent's tests' is not suitable as an acceptance control: mutations catch what a judge, relying on text, misses.

## What It Cost

All of K1 ran on CPU, taking 1.7 hours for the longest product, with products processed in parallel. K1b cost $12.51: blind writer $6.51 (3.49 million input tokens, pre-run estimate was $3-6), mutant variant $5.98, judges $0.02.

## Limitations

- 47 ACs were excluded from the sample: 19 with fewer than three mutation locations, 18 with tests requiring an external service (OIDC), 10 with runs that were red in our environment due to manual DB state. This biases the sample towards tests that run in a clean environment.
- Confirmation of non-equivalence was obtained using a targeted test with knowledge of the mutant, not blindly, and after intermediate data. 16 ACs remained unanswered.
- There is no blind human verification. LLM judges do not replace it, and this result is negative in itself.
- An empty green does not mean the function is broken. The test does not guard it: the next change might break it, and the checkbox will remain.
- The corpus is single: 12 products from one agent on one template.

The agent's 'proven' checkbox is marked honestly by the runner upon a green test run. Approximately one in six criteria, however, is backed by a test that remained green despite every code breakage we introduced.