# A Roadmap to Coding Interview Proficiency

[Back to the coding interview hub](README.md) · [Pattern catalog](pattern-catalog.md) · [Problem-solving framework](problem-solving-framework.md)

This roadmap is a flexible sequence, not a promise that finishing a fixed number of problems will clear an interview. A useful target is about 12 weeks at roughly 8-12 focused hours per week; adjust the pace to your baseline and schedule. If fundamentals are already strong, move faster through review and spend the saved time on mixed, timed practice.

## Before week 1: baseline and setup

- Choose one interview language you can write comfortably without relying on autocomplete. Review its maps, sets, queues, heaps, sorting, and common string/array operations.
- Complete two unseen easy/medium problems in interview conditions and record what slowed you down.
- Categorize each gap as knowledge, modeling, pattern choice, proof, implementation, complexity, or testing.
- Schedule realistic study blocks. Consistency and review are more useful than long sessions that you cannot sustain.

## The 12-week sequence

| Week | Focus | Evidence of progress |
| --- | --- | --- |
| 1 | Big-O, arrays, strings, hash maps/sets | Explain a baseline and a one-pass lookup/counting solution; analyze expected map costs |
| 2 | Prefix sums, two pointers, sliding windows | Distinguish contiguous from non-contiguous problems; explain why a window is or is not monotone |
| 3 | Linked lists, stacks, queues, binary search, intervals | Trace pointer changes; write a boundary-safe binary search and state its invariant |
| 4 | Recursion, backtracking, subsets/permutations | Define recursive state and termination; prune only provably impossible branches |
| 5 | Trees, traversals, BSTs, recursion on trees | Choose preorder/inorder/postorder/BFS based on the information needed |
| 6 | Heaps, tries, DSU, sorting/sweep line | Select a structure from required operations; compare heap selection with full sorting |
| 7 | Graph modeling, DFS/BFS, components, grids | Convert a story or grid into nodes/edges; handle disconnected and cyclic graphs |
| 8 | Topological sort, shortest paths, weighted graphs | Detect cycles in dependency ordering; use BFS only for unweighted shortest paths and Dijkstra only with nonnegative weights |
| 9 | DP fundamentals, one-dimensional and grid DP, knapsack | Define state, recurrence, base cases, evaluation order, and final answer before coding |
| 10 | Sequence/string/interval DP, greedy proof, bit/math review | Distinguish DP from greedy and justify the selected proof strategy |
| 11 | Mixed unseen problems under time limits | Select a technique without a topic label and communicate trade-offs while solving |
| 12 | Mock interviews, targeted repair, final revision | Complete realistic mock sessions; fix recurring gaps and re-solve missed problems from scratch |

Keep the [pattern catalog](pattern-catalog.md) open while learning a topic, but close it during mixed practice. For DP, follow the deeper [state-design guide](dynamic-programming.md).

## A repeatable weekly practice loop

Aim to touch both learning and retrieval each week:

- **Learn:** review one concept and explain it in your own words.
- **Model:** trace a worked example; write its invariant or recurrence.
- **Practice:** attempt representative problems from easy to medium without topic labels.
- **Review:** revisit previous misses and solve a nearby variant.
- **Simulate:** do at least one timed session and narrate your reasoning.

One possible session (about 85-90 minutes):

1. 10 minutes: retrieve the key idea from memory; write the invariant or state definition.
2. 35-40 minutes: solve one problem without viewing a solution.
3. 15 minutes: test edge cases and analyze complexity.
4. 25 minutes: compare against a trusted explanation, note the gap, and summarize the approach in a few sentences.

If stuck, give yourself time to make a real attempt. Use a hint or explanation as feedback, not as a substitute for re-solving. Schedule a blank-page re-solve later (for example, the next day and again the next week); space intervals to fit your retention and calendar.

## Keep an error log

After every meaningful practice problem, capture:

| Field | Example |
| --- | --- |
| Problem and date | Link/title and attempt date |
| First approach | Brute force or initial hypothesis |
| Actual key property | E.g. “window is contiguous and each index moves forward at most once” |
| What failed | E.g. assumed sums are monotone despite negative input |
| Correctness argument | Invariant, recurrence, exchange argument, or graph-order property |
| Complexity | Include auxiliary space and output where relevant |
| Next review | A date to re-solve without notes |

Classify errors; a solved question without a clear reason for the miss provides little guidance on what to study next.

## Choose a practice set

- Use one curated set such as [Grind 75](https://www.techinterviewhandbook.org/grind75/) or [LeetCode Top Interview 150](https://leetcode.com/studyplan/top-interview-150/) to select representative problems.
- Do not try to finish every list simultaneously. A shorter set you understand and can revisit is more useful than shallow exposure to multiple lists.
- Use the starting problems linked from each section of the [pattern catalog](pattern-catalog.md); add unfamiliar variations when a pattern feels memorized rather than understood.
- Read official/editorial solutions after your attempt to identify a missing insight, then re-derive the solution independently.

## If you have less time

### Four weeks

Spend the first half on complexity, arrays/strings/maps, pointers/windows, binary search, stacks, and trees. Cover graph traversal, heaps, intervals, and basic DP next. Use the remaining time for mixed practice and mocks. Prioritize a correct explanation and coverage of common patterns over deep specialty algorithms.

### One week

Do not attempt to learn every topic from scratch. Review your strongest foundations, repair the biggest known gaps, practice a small representative mix, and complete realistic mock sessions. Revisit language/library details and the interview loop. Be honest about topics that need more study.

## Readiness checks

Before scheduling or taking interviews, test yourself against unseen problems:

- Can you clarify requirements and generate useful edge cases before coding?
- Can you describe a brute-force solution, then justify why an improved method fits?
- Can you select a structure by its operations and analyze its costs accurately?
- Can you explain an invariant, recurrence, or proof idea in plain language?
- Can you implement and test a solution in the interview language without a walkthrough?
- Can you recover when the first approach fails and communicate uncertainty productively?
- Can you handle a timed mock with a different prompt and no pattern label?

If one answer is consistently “no,” target that skill rather than increasing total question count.

## Interview performance is more than the algorithm

Practice the full interaction:

1. Restate the task and ask clarifying questions.
2. Agree on examples and edge cases.
3. Explain a baseline and propose an improvement.
4. State time/space costs and why the approach is correct.
5. Code while communicating decisions.
6. Trace tests, fix issues, and mention limitations.

The [Tech Interview Handbook's interview cheatsheet](https://www.techinterviewhandbook.org/coding-interview-cheatsheet/) and [coding-interview preparation guide](https://www.techinterviewhandbook.org/coding-interview-prep/) provide additional advice on evaluation, communication, language choice, preparation, and mock interviews.
