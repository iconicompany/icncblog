---
title: "The Agent Authored 73% of Commits in 12 Products. For 58% of Changes, We Can Tell Which Criterion They Served"
date: '2026-10-28'
description: "Census of an autonomous coding agent's corpus: 12 products, 509k lines, 3,143 acceptance criteria, 2,057 commits. Checkmark regressions are rare (2.2%), but the activity log is poorly recoverable: 57.8% of changes are unambiguously linked to a slice or criterion, and history older than a month was erased by transcript rotation. Plus, initial procedure metrics: 34% of commits were made without a passing run after the last change."
tags: ["AI","LLM","Coding Agents","Software Engineering","Research"]
authors: ['ICNC Lab']
language: 'en'
---

An autonomous agent assembled twelve products, most of them over three months. They contain 509 thousand lines of code, 3,143 acceptance criteria (AC), of which 2,368 are marked as proven. Of the 2,057 commits, 1,497 were made by the agent, which is 73%.

On this corpus, we aimed to measure false success: how much "proven" work is actually untested, and whether this can be predicted by how the agent operated. For the second question, a log is needed: which change was made for which criterion. It turned out that for 42% of changes, this cannot be restored.

## What is the Corpus

The products are built on a single template: Bun, Elysia, PostgreSQL, Drizzle. Each specification is broken down into use cases, each with acceptance criteria (AC) and a verification level: unit, integration, browser. Code resides in slices: a slice is a vertical segment of the product with its own code directory and its own ACs. The "proven" checkmark is set by the runner based on a JUnit report, if it finds a passing test with the criterion ID in its name.

The census (K0) was read-only: `git log`, `git show`, `ls-files` in each repository, without checkout. The output contained only aggregates, no code, and no message texts. Execution took about 15 seconds.

| | total |
|---|---|
| products | 12 |
| AC / proven | 3,143 / 2,368 (75%) |
| slices | 133 |
| commits (agent / other) | 2,057 (1,497 / 560) |
| lines in `src` (code / tests) | 508,917 (392k / 117k) |
| ACs ever proven | 2,574 |
| of which lost proof | 57 (2.2%) |

The proportion of proven ACs varies significantly: from 23% for the largest product (682 ACs) to 98-100% for five others. Almost all unproven ACs lack a single test: 765 out of 775. Currently, no criterion has a "test failing" status.

I considered a commit with the trailer `Co-Authored-By: ...Claude` to be an agent's commit. Everything else was attributed to "human", including merge commits, so 560 is an upper bound.

## Almost No Checkmark Regressions

Of the 2,574 ACs proven at least once, 57 lost their checkmark. Almost all transitions were due to "test missing": the test was deleted or renamed. A "test failing" transition was found for one criterion.

In part, this is good news. However, a [parallel measurement (K1)](/en/research/agent-green-tests-that-catch-no-mutants) showed that for 16% of proven ACs, the associated test catches no code mutations. A test that checks nothing will also not turn red. Therefore, the low proportion of regressions primarily indicates that the runner does not detect failures.

## Unit of Analysis Differs from Plan

The plan required at least 2,000 slices to train a false success predictor. There are 133 slices, 137 throughout history. Even commits number only 2,057. The threshold was calculated based on "slice runs", but in these products, a slice is a module the size of a use case, and it's not run hundreds of times.

The unit of analysis becomes the acceptance criterion: 3,143, of which 2,574 were proven. There are 742 "agent commit x slice" pairs, which can be used as a substitute for an episode.

## Transcripts: What Was Preserved

Claude Code automatically deletes old session transcripts (`cleanupPeriodDays` setting). By the time of the census, project directories contained sessions from 2026-10-12; three products had no transcripts left at all. Part of the history from 2026-08-27 was found in an archived copy of the home directory. In total for K2, 375 main sessions were gathered for 2026-08-27 - 2026-10-27 across 11 products.

Takeaway for anyone planning to study their agent: `cleanupPeriodDays` in Claude Code settings should be increased *before* history is needed. We copied the transcripts to an archive (2.1 GB) only after some of the history had already disappeared.

Of 11,229 tool calls in the preserved sessions, 86% were to Bash. The agent modifies files via `sed`, heredoc, and scripts more often than through the built-in editor, which is the main difficulty for the parser.

## K2: Log Parser

The parser extracts events from transcripts: file modification, test execution and its output, commit, verification calls. For modifications via Bash, it parses diffs, VCS commands, or uses command-based heuristics. The output includes only paths, types, criterion IDs, hashes, and timestamps.

375 sessions yielded 17,374 events, of which 12,025 were modifications; 6,499 modifications were to `src` and tests within products. Each modification needs to be linked to a slice, or preferably to a criterion.

| attachment level | proportion of src/test modifications |
|---|---|
| strict: slice path or criterion ID in modification | 49.0% |
| + via commit the modification was part of | **57.8%** |
| + "between two modifications of the same slice" | 71.1% |
| + any unique context | 78.7% |
| to a specific AC (strictly) | 26.8% |

The threshold was 80% unambiguous attachment. Strict attachment combined with the commit yields 57.8%. A sample of 30 events was cross-referenced with the original transcript; Claude verified it, not a human. Among them, 16 events had strict attachment or attachment via commit; all 16 were correctly attached to a slice, and for 10 of those where a criterion was named, it was also correct. Another 6 events were attached only by context, 3 of which were incorrect, so contextual levels (up to 78.7%) are not included in the analysis. The remaining 8 were honestly left unattached by the parser.

Half of the unattached modifications pertain to general template boilerplate: gateway, web application, identity module. These changes do not essentially belong to any single slice, and a truthful attachment for them is "nowhere".

## What Can Be Calculated Now

For strictly attached events, some procedure formulas are already computable. These are preliminary metrics: the accuracy of the formulas themselves has not yet been verified, and a failing run is only visible if the agent ran tests via Bash and the output contained failing slices.

| formula | cases | violations |
|---|---|---|
| commit without a passing run after the last modification | 760 | 260 (34%) |
| src slice modification without a failing test for that slice earlier in the session | 2,329 | 1,266 (54%) |
| transition failing → passing, between which only tests were modified | 551 | 125 (23%) |
| acceptance verification without integration or e2e run after modification | 43 | 40 |

The third row is the most interesting. A test failed, the agent modifies the test, the test passes, the code remains untouched. Sometimes this is legitimate: the test was incorrect. But this is precisely the path by which a test that checks nothing emerges. Next step (K3): link this formula to the vacuously green ACs from K1. Of the twelve formulas in the plan, about seven are computable at the slice level; formulas at the criterion level require new transcripts where the agent explicitly invokes acceptance verification.

## Limitations

- Single corpus: one agent, one template. Transferability to other stacks has not been verified.
- "Agent commit" defined by trailer; agent commits without a trailer were attributed to "human".
- Transcripts cover two out of three months and not all products. Subagent sessions were not analyzed in K2.
- Procedure formulas were calculated based on 58% of modifications with reliable attachment. If the unattached 42% behave differently (e.g., boilerplate modifications are made without tests more often), the estimates are biased.
- Attachment accuracy was checked on 30 events, and Claude verified them, not a human. There is no independent human labeling.
- The corpus is live: sessions continue to run in the products, and a repeat run yields slightly different numbers. Snapshot as of 2026-10-27.

The agent produces many checkmarks, and they are rarely reverted. But what it did for each of them, according to its own log, can only be determined for six out of ten modifications.