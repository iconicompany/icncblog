---
title: "Agent Wrote 73% of Commits in 12 Products. For 58% of Edits, It's Possible to Determine Which Criterion Was Addressed"
date: '2026-10-28'
description: "Census of an autonomous coding agent's corpus: 12 products, 509K lines, 3,143 acceptance criteria, 2,057 commits. Checkbox regressions are rare (2.2%), but the activity log is poorly recoverable: 57.8% of edits are unambiguously tied to a slice or criterion, and history older than a month was erased by transcript rotation. Plus the first procedure counters: 34% of commits were made without a green run after the last edit."
tags: ["AI","LLM","Coding Agents","Software Engineering","Research"]
authors: ["ICNC Lab"]
language: "en"
---

The autonomous agent developed twelve products, most of them over three months. They contain 509 thousand lines of code, 3,143 acceptance criteria, of which 2,368 are marked as proven. Of 2,057 commits, 1,497 were made by the agent, which is 73%.

On this corpus, we aimed to measure false success: how much of what is "proven" is actually untested, and whether this can be predicted by how the agent operated. The second question requires a log: which edit was made for which criterion. It turned out that for 42% of edits, this cannot be recovered.

## What is the Corpus

The products are built on a single template: Bun, Elysia, PostgreSQL, Drizzle. Each specification is broken down into use cases, each with acceptance criteria (AC) and a verification level: unit, integration, browser. The code lives in slices: a slice is a vertical piece of a product with its own code directory and its own ACs. The "proven" checkbox is set by the runner based on a JUnit report if it finds a green test with the criterion's ID in its name.

The census (K0) was read-only: `git log`, `git show`, `ls-files` in each repository, without checkout. The output consisted only of aggregates, no code or message texts. The run took about 15 seconds.

| | total |
|---|---|
| products | 12 |
| AC / proven | 3,143 / 2,368 (75%) |
| slices | 133 |
| commits (agent / others) | 2,057 (1,497 / 560) |
| lines in `src` (code / tests) | 508,917 (392K / 117K) |
| AC, ever proven | 2,574 |
| of these lost proof | 57 (2.2%) |

The proportion of proven ACs varies greatly: from 23% for the largest product (682 ACs) to 98-100% for five others. Almost all unproven ACs lack a single test: 765 out of 775. No criterion currently has a "test red" status.

I considered a commit with the `Co-Authored-By: ...Claude` trailer to be agent-made. All other commits, including merge commits, were attributed to "human," making 560 an upper estimate.

## Almost No Checkbox Regressions

Out of 2,574 ACs proven at least once, 57 lost their checkbox. Almost all transitions were due to "no test": the test was deleted or renamed. A "test fails" transition was found for one criterion.

This is partly good news. However, a [parallel measurement (K1)](/en/research/agent-green-tests-that-catch-no-mutants) showed that for 16% of proven ACs, the associated test does not catch any code mutations. A test that checks nothing will not turn red. Therefore, the low proportion of regressions primarily indicates that the runner does not detect failures.

## Unit of Analysis Is Not What Was Planned

The plan required at least 2,000 slices to train a false success predictor. There are 133 slices, 137 throughout history. Even commits are only 2,057 in total. The threshold was calculated for "slice runs," but in these products, a slice is a module the size of a use case; it's not run hundreds of times.

The unit of analysis becomes the acceptance criterion: 3,143, of which 2,574 were proven. There are 742 "agent commit x slice" pairs, which can be used as a substitute for an episode.

## Transcripts: What Was Preserved

Claude Code stores session transcripts for 30 days and deletes older ones. By the time of the census, project directories contained sessions from 2026-10-12; three products had no transcripts left at all. Part of the history from 2026-08-27 was found in an archived copy of the home directory. In total for K2, 375 main sessions from 2026-08-27 to 2026-10-27 were collected for 11 products.

The takeaway for anyone intending to study their agent: `cleanupPeriodDays` in Claude Code settings should be increased before the history is needed. We copied transcripts to an archive (2.1 GB) only after part of the history was already lost.

Of 11,229 tool calls in the preserved sessions, 86% were to Bash. The agent edits files via `sed`, heredoc, and scripts more often than through the built-in editor, which is the main difficulty for the parser.

## K2: Log Parser

The parser extracts events from transcripts: file edits, test runs and their results, commits, and verification calls. For edits via Bash, it parses the diff, VCS command, or heuristics based on the command. The output only includes paths, types, criterion IDs, hashes, and timestamps.

375 sessions yielded 17,374 events, including 12,025 edits; 6,499 edits to `src` and tests within products. Each edit needs to be linked to a slice, or preferably to a criterion.

| linkage level | share of src/test edits |
|---|---|
| strict: slice path or criterion ID in edit | 49.0% |
| + via commit containing the edit | **57.8%** |
| + "between two edits of the same slice" | 71.1% |
| + any unique context | 78.7% |
| to a specific AC (strictly) | 26.8% |

The threshold was 80% unambiguous linkage. Strict linkage, combined with the commit, yields 57.8%, and in manual verification of 30 events, it made no mistakes: 16 out of 16 to a slice, 10 out of 10 to a criterion. Contextual levels raise the share to 78.7%, but they erred in 3 out of 6 cases during verification, so they are not included in the analysis.

Half of the unlinked edits are related to the general template boilerplate: gateway, web application, identification module. These edits are not essentially tied to any single slice, and a truthful linkage for them is "nowhere."

## What Can Be Calculated Now

For strictly linked events, some procedural formulas are already computable. These are preliminary counters: the accuracy of the formulas themselves has not yet been verified, and a red run is only visible if the agent executed tests via Bash and failed slices were found in the output.

| formula | instances | violations |
|---|---|---|
| commit without a green run after the last edit | 760 | 260 (34%) |
| `src` slice edit without a red test for that slice earlier in the session | 2,329 | 1,266 (54%) |
| red → green transition where only tests were edited in between | 551 | 125 (23%) |
| acceptance verification without integration or e2e run after edit | 43 | 40 |

The third row is the most interesting. A test failed, the agent edits the test, the test is green, the code is untouched. Sometimes this is legitimate: the test was incorrect. But this is precisely how a test that checks nothing comes into existence. The next step (K3): link this formula to the empty green ones from K1. Of the twelve formulas in the plan at the slice level, about seven are computable; formulas at the criterion level require new transcripts where the agent explicitly invokes acceptance verification.

## Limitations

- Single corpus: one agent, one template. Transferability to other stacks was not verified.
- "Agent commit" is defined by the trailer; agent commits without a trailer were counted as "human."
- Transcripts cover two out of three months and not all products. Sub-agent sessions were not analyzed in K2.
- Procedural formulas are calculated based on 58% of edits with reliable linkage. If the unlinked 42% are structured differently (e.g., boilerplate edits are made without tests more often), the estimates are biased.
- The corpus is live: sessions continue to run in the products, and a repeat run yields slightly different numbers. Snapshot as of 2026-10-27.

The agent has many checkboxes, and they are rarely reverted. However, what it did for each of them can only be determined from its own log for six out of ten edits.