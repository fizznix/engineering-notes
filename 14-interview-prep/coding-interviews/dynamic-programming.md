# Dynamic Programming: Design the State, Then the Algorithm

[Back to the coding interview hub](README.md) · [Problem-solving framework](problem-solving-framework.md) · [Pattern catalog](pattern-catalog.md)

Dynamic programming (DP) is useful when a problem can be decomposed into smaller states, those states overlap, and each state can be computed from a manageable set of smaller states. Memoization and tabulation are evaluation strategies; neither one substitutes for a correct state definition.

The hardest part is usually not writing a table. It is proving that the state captures everything future decisions need.

## The six design questions

Write answers to these questions before choosing a table shape:

1. **State:** What exact subproblem does `dp[...]` represent?
2. **Choice:** What decisions are available from this state?
3. **Transition:** How does each choice lead to smaller states?
4. **Base cases:** What are the smallest states whose answers are known directly?
5. **Evaluation order:** Which states must be computed before this one?
6. **Final answer:** Which state or aggregation of states matches the question?

Then establish the proof:

- Every valid solution must fall into one of the recurrence's cases.
- Each case's subproblem result is sufficient to construct or score a valid solution.
- The base cases are correct.
- The evaluation order computes dependencies before use.

If asking for the actual path, subset, or sequence, store choices or reconstruct them from the table; an optimal value alone may not preserve the requested witness.

## Example: House Robber

Given a row of houses with nonnegative values, choose a subset with no adjacent houses and maximize the sum.

Define `best[i]` as the maximum amount from the first `i` houses. For the next house, either skip it (`best[i - 1]`) or take it and skip its neighbor (`value[i - 1] + best[i - 2]`):

```text
best[0] = 0
best[1] = value[0]
best[i] = max(best[i - 1], value[i - 1] + best[i - 2])
```

The cases are exhaustive because any valid solution either takes the last house or does not. In the first case, the previous house cannot be taken; in the second, the first `i - 1` houses remain available. Thus the recurrence is correct if its base cases are correct. A rolling pair of values reduces auxiliary space from `O(n)` to `O(1)`.

## Recognizing families of DP states

These are related shapes, not a list that every problem must match.

| Family | Typical state shape | Key modeling question | Representative problem |
| --- | --- | --- | --- |
| One-dimensional / take-or-skip | Position `i` | What choices include or exclude the current item? | [House Robber](https://leetcode.com/problems/house-robber/) |
| Grid paths | Cell `(r, c)` | Which predecessor cells can reach this cell? | [Unique Paths](https://leetcode.com/problems/unique-paths/) |
| 0/1 knapsack / subset sum | Item index and remaining capacity/target | Can each item be used once, and what are include/exclude cases? | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) |
| Unbounded choice / coin change | Remaining amount, sometimes plus item index | Can a choice be reused? Does order of choices matter to the count? | [Coin Change](https://leetcode.com/problems/coin-change/), [Coin Change II](https://leetcode.com/problems/coin-change-ii/) |
| Two-sequence DP | Prefix lengths `(i, j)` | If current symbols match, or do not, how do the prefixes relate? | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) |
| Increasing subsequence | Position, often with a best length ending there | Which earlier positions can precede this item? | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) |
| Interval DP | Substring/subarray bounds `(l, r)` | Which split or endpoint choice partitions the interval into smaller intervals? | [Minimum Score Triangulation of Polygon](https://leetcode.com/problems/minimum-score-triangulation-of-polygon/) |
| String partitioning | Prefix endpoint or position plus auxiliary state | Where can the last valid segment begin? | [Word Break](https://leetcode.com/problems/word-break/) |

## A reliable problem-solving progression

1. **Start with recursion.** Express the answer using choices and smaller instances. State clearly what the function means.
2. **Draw the call tree on a tiny input.** Repeated identical arguments are evidence that memoization may help.
3. **Memoize the complete state.** If the result also depends on remaining capacity, prior choice, or another constraint, that information belongs in the state.
4. **Analyze state count and transition cost.** A table with `S` states and `T` work per state costs `O(S * T)`, plus storage.
5. **Convert to bottom-up only when dependencies are clear.** Choose an order that computes prerequisites first; topological order is the general principle.
6. **Optimize memory only after validating the recurrence.** Keep only states still needed; for knapsack, loop direction can change whether an item is reusable.

Memoized recursion is often easier to derive. Tabulation may avoid recursion limits and make time/space explicit. They should implement the same recurrence.

## Subtle distinctions that cause bugs

### 0/1 vs. unbounded knapsack

If each item can be used once, a one-dimensional capacity table is commonly updated from high capacity to low so the current item is not reused during the same iteration. If items may be reused, update from low to high. The direction is a consequence of the dependency relation, not a memorized trick; derive it from the table's meaning.

### Counting combinations vs. permutations

For coin-change counting, placing the coin loop outside the amount loop commonly counts combinations (order does not matter); placing the amount loop outside can count ordered sequences. State what is being counted and choose iteration order accordingly.

### State compression vs. reconstructing a solution

Rolling arrays may preserve the optimum value but discard enough history to reconstruct the choices. Keep parent/choice information when the output requires the selected path or items.

### DP vs. greedy

DP compares all relevant decisions represented by the state. Greedy takes one decision and needs a proof that it is safe. A greedy-looking local choice is not justified just because it works on a few examples.

## Common failure modes

- Starting with a table before defining what an entry means.
- Omitting a variable from the state even though it changes future choices.
- Using a recurrence that misses a valid case or counts a case multiple times.
- Confusing subsequence (not necessarily contiguous) with substring/subarray (contiguous).
- Incorrect loop direction for item reuse.
- Optimizing memory before verifying the full recurrence.
- Calling every recursion problem DP, even when subproblems do not overlap.

## Suggested practice order

1. [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) - base cases and one-dimensional recurrence.
2. [House Robber](https://leetcode.com/problems/house-robber/) - take/skip reasoning.
3. [Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/) - optimize a recurrence.
4. [Unique Paths](https://leetcode.com/problems/unique-paths/) - grid state.
5. [Coin Change](https://leetcode.com/problems/coin-change/) - unbounded minimum.
6. [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) - 0/1 subset feasibility.
7. [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) - two sequences.
8. [Word Break](https://leetcode.com/problems/word-break/) - prefix partitioning.
9. [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) - sequence state and an optimization follow-up.
10. [Edit Distance](https://leetcode.com/problems/edit-distance/) - two-dimensional string transformation.

Attempt to define the state and recurrence before reading an editorial. After studying a solution, close it and derive the proof and implementation again.

## Further reading

- [AlgoMaster: 20 Patterns to Master Dynamic Programming](https://blog.algomaster.io/p/20-patterns-to-master-dynamic-programming) - broad pattern taxonomy and practice links.
- [Tech Interview Handbook: Dynamic Programming study guide](https://www.techinterviewhandbook.org/algorithms/dynamic-programming/) - linked from the handbook's [DSA study cheatsheets](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/); the individual page was not retrievable during research.
- [Princeton Algorithms: fundamentals](https://algs4.cs.princeton.edu/10fundamentals/) - complexity and mathematical foundations for algorithms.
