# Coding Interview Pattern Catalog

[Back to the coding interview hub](README.md) · [Foundations](foundations.md) · [Problem-solving framework](problem-solving-framework.md) · [Dynamic programming](dynamic-programming.md)

Use this catalog to generate candidate approaches. A cue is not sufficient by itself: confirm the preconditions, maintain the stated invariant, and test a counterexample. Complexity is typical for the indicated approach; include output size and representation costs where relevant.

## Sequences, arrays, and strings

| Pattern | When it fits | Core idea / correctness handle | Starting problems |
| --- | --- | --- | --- |
| Hashing and frequency counts | Membership, complements, duplicates, counts, or grouping by a key | Store exactly the information needed for future lookups; explain expected map cost | [Two Sum](https://leetcode.com/problems/two-sum/), [Group Anagrams](https://leetcode.com/problems/group-anagrams/) |
| Prefix sum + map | Many range sums or counting subarrays with a target sum; values may be negative | `prefix[j] - prefix[i]` is the sum of an interval; count earlier prefix values that make the difference equal the target | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/), [Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable/) |
| Two pointers | Sorted arrays, opposite ends, partitioning, or in-place compaction | Each move must safely eliminate candidates; total time is often `O(n)` when each pointer moves only forward | [Two Sum II](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/), [3Sum](https://leetcode.com/problems/3sum/) |
| Sliding window | A contiguous subarray/substring with a maintainable condition or fixed length | Expand one boundary and repair/shrink the other; prove no discarded start can later improve the answer | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/), [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) |
| Binary search | Sorted data, or a monotone yes/no predicate over possible answers | Preserve a range containing the boundary; be explicit about the search invariant and termination | [Binary Search](https://leetcode.com/problems/binary-search/), [First Bad Version](https://leetcode.com/problems/first-bad-version/) |
| Monotonic stack / deque | Next greater/smaller element, unresolved intervals, or sliding-window extrema | Keep candidates in monotone order; each item enters and leaves at most once, giving amortized `O(n)` | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/), [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) |
| Sorting and sweep line | Ordering simplifies pair comparisons, interval overlap, or event processing | After sorting, prove that the processed prefix is complete and future items cannot affect it | [Merge Intervals](https://leetcode.com/problems/merge-intervals/), [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) |

### Important boundary: sliding window is not a universal subarray trick

The standard shrink-while-invalid window relies on being able to repair a window by moving its left boundary forward. For a sum threshold with nonnegative elements, extending right cannot lower the sum and shrinking left cannot raise it. With negative elements, that monotonicity fails. For exact-sum counting with negatives, prefix sums plus a frequency map is often the right model instead.

## Linked lists, stacks, queues, and heaps

| Pattern | When it fits | Core idea / correctness handle | Starting problems |
| --- | --- | --- | --- |
| Fast and slow pointers | Cycles, midpoint, or a repeated state with different traversal speeds | A meeting implies a cycle in a finite list; reason about pointer speed and relative distance | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/), [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list/) |
| In-place list rewiring | Reverse a list or part of it without allocating replacement nodes | Save the next reference before overwriting a link; keep clear previous/current/next roles | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/), [Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group/) |
| Stack matching / monotonic stack | Nested syntax, undo history, or a next-greater/smaller relation | The top is the only unresolved candidate that can be affected by the next item | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/), [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) |
| Heap / top `k` | Repeatedly retrieving an extreme, merging sorted streams, or keeping the best `k` | A heap maintains only a partial order; a size-`k` heap gives `O(n log k)` selection | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/), [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) |

## Trees and recursive structures

| Pattern | When it fits | Core idea / correctness handle | Starting problems |
| --- | --- | --- | --- |
| DFS recursion / postorder | The answer for a node depends on answers from its children | Define exactly what each recursive call returns; combine child results once | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/), [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) |
| BFS / level order | Level-by-level processing or shortest path in a tree by edge count | Queue order processes nodes in nondecreasing depth | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) |
| BST ordering | Queries or validation that depend on the global ordering property | Pass valid lower/upper bounds; checking only parent-child comparisons is insufficient | [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/), [Lowest Common Ancestor of a BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) |
| Trie | Prefix queries, word lookup, or dictionary-based search | Follow one edge per symbol; query cost is `O(L)` but memory depends on stored prefixes | [Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/) |

Tree recursion is usually simpler than graph recursion because a tree has no cycles and each child has one parent. General graph traversal needs a visited set.

## Graphs and connectivity

| Pattern | When it fits | Core idea / correctness handle | Starting problems |
| --- | --- | --- | --- |
| BFS | Reachability or shortest path by edge count in an unweighted graph | FIFO order explores nondecreasing distance; mark on enqueue to avoid duplicate work | [Number of Islands](https://leetcode.com/problems/number-of-islands/), [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) |
| DFS | Components, reachability, cycle checks, or exploring a graph structure | Mark visited before exploring neighbors; account for directed vs. undirected edges | [Clone Graph](https://leetcode.com/problems/clone-graph/), [Number of Provinces](https://leetcode.com/problems/number-of-provinces/) |
| Topological order | Prerequisite/dependency relations that should form a DAG | Repeatedly remove zero-indegree vertices, or use DFS finish order; fewer output vertices than graph vertices indicates a cycle | [Course Schedule](https://leetcode.com/problems/course-schedule/), [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) |
| Disjoint-set union | Connectivity as edges/groups are incrementally joined | `find` identifies a component; `union` merges roots; path compression and size/rank keep trees shallow | [Redundant Connection](https://leetcode.com/problems/redundant-connection/), [Number of Provinces](https://leetcode.com/problems/number-of-provinces/) |
| Dijkstra | Shortest paths with nonnegative edge weights | A min-heap finalizes the currently closest unsettled vertex; reject stale heap entries if using lazy updates | [Network Delay Time](https://leetcode.com/problems/network-delay-time/), [Path With Minimum Effort](https://leetcode.com/problems/path-with-minimum-effort/) |

Modeling matters: grids are implicit graphs; word transformations can be graphs of states; a dependency list becomes a directed graph. A graph may be disconnected, and an adjacency-list traversal costs `O(V + E)`. Do not use Dijkstra when negative edge weights are allowed.

## Search spaces, choices, and optimization

| Pattern | When it fits | Core idea / correctness handle | Starting problems |
| --- | --- | --- | --- |
| Backtracking | Need to enumerate or test constrained combinations, permutations, or paths | Choose, recurse, undo; pruning must remove only branches that cannot satisfy the goal | [Subsets](https://leetcode.com/problems/subsets/), [Combination Sum](https://leetcode.com/problems/combination-sum/) |
| Greedy | A locally preferred choice appears to simplify the remaining problem | Prove safety, often via an exchange argument or a stays-ahead argument; intuition alone is not enough | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/), [Jump Game](https://leetcode.com/problems/jump-game/) |
| Dynamic programming | Subproblems overlap and a solution can be assembled from smaller states | Define state, recurrence, base cases, order, and answer; see the [DP guide](dynamic-programming.md) | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/), [Coin Change](https://leetcode.com/problems/coin-change/) |
| Divide and conquer | Split into mostly independent smaller instances, solve them, and combine | Prove the split covers the original problem and account for combine cost | [Sort an Array](https://leetcode.com/problems/sort-an-array/), [Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) |

## A decision aid, not a lookup table

| Problem property | Candidate technique | Verify before committing |
| --- | --- | --- |
| Need membership or counts over a scan | Hash map / set | Does expected-time hashing fit? Is an ordered answer needed? |
| Contiguous range and one endpoint can move monotonically | Sliding window | Does validity stay maintainable when the left edge moves? |
| Sorted order and pair elimination | Two pointers | Does each pointer move discard only impossible candidates? |
| Monotone feasible / infeasible answers | Binary search on answer | Can you prove one transition and write a correct feasibility test? |
| Optimal result depends on smaller repeated states | DP | Can the complete state be defined without losing information? |
| Small choice tree, require all valid answers | Backtracking | Can constraints prune branches without skipping valid results? |
| Repeated minimum/maximum selection | Heap | Is partial ordering enough, or must all values be sorted? |
| Reachability / dependencies / connectivity | Graph algorithm | Are edges directed or weighted? Is the graph acyclic? |
| Local choice that seems best | Greedy | Can you prove an optimal solution exists that begins with this choice? |

For a repeatable method to test these candidate fits, read the [problem-solving framework](problem-solving-framework.md).
