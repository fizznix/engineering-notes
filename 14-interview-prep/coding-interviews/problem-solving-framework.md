# A Framework for Solving Coding Interview Problems

[Back to the coding interview hub](README.md) · [Pattern catalog](pattern-catalog.md) · [Roadmap](roadmap.md)

Patterns are not labels to guess from a prompt. They are reusable algorithmic ideas whose assumptions match a problem's structure. Use them as hypotheses, then verify the assumptions and explain why the resulting algorithm is correct.

## The interview loop

### 1. Clarify and model

Restate the goal and ask only questions that can change the algorithm or output:

- Can the input be empty? Can values repeat or be negative?
- Is the input sorted? Is it mutable?
- Is the graph directed, weighted, disconnected, or allowed to contain cycles?
- Does the output require one answer, all answers, the count, or an actual path/sequence?
- What are the bounds on input size, values, and result size?

Create a small normal example, a boundary example, and an example designed to break a tempting assumption. For a string-window problem, include repeated characters; for a sum problem, include negative values if permitted.

### 2. Establish a baseline

Describe the simplest correct approach before optimizing. Examples include checking every pair (`O(n^2)`) or enumerating all subsets (`O(2^n)`). A baseline reveals which work is repeated and gives a correctness reference.

Do not present an incorrect shortcut as a baseline. If the task asks for all answers, returning the first matching answer is not a correct baseline.

### 3. Use constraints to narrow possibilities

Estimate the number of operations a candidate approach performs. Identify a lower bound when possible: if every element may determine the answer, reading all `n` elements costs `Omega(n)`. If the output itself has `k` items, writing it costs `Omega(k)`.

The goal is not always to prove the globally optimal algorithm. It is to understand why a candidate fits the input limits and what trade-off it makes.

### 4. Match structure to technique

Describe the underlying shape:

- **Sequence with repeated membership or counts:** a map or set may remove repeated scans.
- **Contiguous range:** consider prefix sums or a sliding window; negative values can invalidate the usual monotonic-window argument.
- **Sorted order or monotone predicate:** binary search or two pointers may eliminate candidates.
- **Next greater/smaller element:** a monotonic stack may store only unresolved candidates.
- **Repeated best/next item:** a heap may avoid re-sorting everything.
- **Tree or graph reachability:** DFS/BFS; choose BFS for shortest paths when all edges have equal cost.
- **Dependencies:** topological ordering applies only when the dependency graph is acyclic.
- **Small choice space with constraints:** backtracking, ideally with sound pruning.
- **Overlapping subproblems and optimal substructure:** dynamic programming may reuse subproblem results.
- **A locally optimal choice that must remain globally safe:** consider greedy, then prove why it is safe.

For more signals and counterexamples, see the [pattern catalog](pattern-catalog.md).

### 5. State the reason it is correct

Use the proof idea that fits the approach:

- **Loop invariant:** state what is true before/after every iteration and show initialization, preservation, and termination.
- **Induction / recursion:** show base cases are correct and each answer is built from smaller correct answers.
- **BFS layers:** after processing distance `d`, all reachable vertices at distance at most `d` have been handled; unit edge costs make first discovery shortest.
- **Greedy exchange argument:** show that replacing a choice in an optimal solution with the greedy choice does not make it worse.
- **DP decomposition:** every feasible solution is covered by the recurrence's cases, and each case combines valid subproblem solutions.
- **Monotone binary search:** show the predicate has one transition and the search preserves a range containing the boundary.

A compact explanation is enough if it is precise. Naming a pattern is not a proof.

### 6. Implement, test, and communicate

Before typing, state the chosen approach and expected complexity. Keep the implementation aligned with the explanation. Then trace one example and test:

- Empty and smallest permitted input.
- One element, duplicates, or all elements equal.
- Boundary values and overflow-sensitive arithmetic.
- Already sorted, reverse sorted, or highly skewed structure.
- No solution, one solution, and multiple solutions if applicable.
- Cycles/disconnected nodes for graphs; repeated characters for string problems.

Mention any simplification or assumption to the interviewer. See the [roadmap](roadmap.md) for a practice routine that trains this communication rather than code alone.

## Example: longest substring without repeated characters

Suppose the task is to find the length of the longest contiguous substring with no repeated character.

1. **Model:** the substring must be contiguous. Repeated characters invalidate a candidate window.
2. **Baseline:** enumerate every start and extend while tracking seen characters. In the worst case, this takes `O(n^2)` time.
3. **Candidate:** maintain a window `[left, right]` and the most recent index for each character.
4. **Update:** when the character at `right` was last seen inside the current window, move `left` just past that occurrence. Otherwise it can stay. Record the longest width.
5. **Invariant:** after processing each right endpoint, the window contains no repeated character. `left` only moves forward, so each index is processed at most a constant number of times.
6. **Cost:** expected `O(n)` time and `O(min(n, alphabet size))` map space, depending on the character model.
7. **Tests:** empty string, all distinct characters, all repeated characters, and a repeat just outside the current window.

The central reasoning is not “substring means sliding window.” It is that the window is contiguous and that invalidity can be repaired by moving its left boundary forward without reconsidering earlier starts.

## Distinguish commonly confused techniques

| Techniques | Distinguishing question |
| --- | --- |
| Sliding window vs. prefix sum + hash map | Can the window condition be maintained monotonically as the left edge moves? For exact-sum subarrays with negative values, use prefix sums and counting rather than the standard sum-shrinking window. |
| Two pointers vs. binary search | Does moving one pointer eliminate a region of candidates using sorted order or a dominance argument? If a single monotone yes/no predicate partitions a search range, binary search may fit better. |
| DFS vs. BFS | Is any traversal enough, or is the shortest path by number of edges required? BFS gives the latter for unweighted graphs. |
| Greedy vs. DP | Can each local choice be proven safe for an optimal solution? If earlier choices affect later possibilities and no such proof is available, a DP state may be needed. |
| Backtracking vs. DP | Are we enumerating distinct feasible choices, or do branches repeatedly reach the same state and ask for the same subproblem answer? In the latter case, memoization may help. |
| Heap vs. sorting | Do we need the entire order once, or only repeated access to the current best `k`/next item? A heap can avoid sorting everything. |
| DP vs. divide and conquer | Do recursive branches overlap? Memoization helps when repeated states exist; divide and conquer is useful when subproblems are distinct and independently combined. |

## When you are stuck

1. Rephrase the problem as a concrete operation or state transition.
2. Draw a small input or trace the manual process.
3. Write the brute-force search space and identify its cost.
4. Ask what information would let you avoid revisiting candidates.
5. Enumerate applicable structures (set, map, stack, queue, heap, tree, graph) and what operation each accelerates.
6. Change one assumption in a counterexample to test whether the idea still works.
7. If no better approach is justified, communicate the correct baseline and its limitation.

These are starting moves, not a mechanical checklist. The [Tech Interview Handbook's techniques](https://www.techinterviewhandbook.org/coding-interview-techniques/) also recommends drawing, hand-solving, generating examples, decomposing the problem, and looking for repeated work.
