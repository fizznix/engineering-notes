# Foundations and Data Structures for Coding Interviews

[Back to the coding interview hub](README.md) · [Pattern catalog](pattern-catalog.md)

This is a working refresher, not a replacement for a full algorithms course. The interview skill is to connect an operation the problem needs to a representation that can perform it efficiently, while accounting for the representation's limitations.

## Start with the cost model

Let `n` be the input size. A useful complexity analysis explains how work or memory grows with `n`; it does not need to predict exact seconds.

| Growth | Typical example | Practical implication |
| --- | --- | --- |
| `O(1)` | Array index access | Does not grow with input size |
| `O(log n)` | Binary search on a sorted range | Each step discards a constant fraction |
| `O(n)` | One full scan | Usually the natural lower bound when every item may matter |
| `O(n log n)` | Comparison sorting | Common cost of ordering arbitrary values |
| `O(n^2)` | Compare every pair | May work only when constraints are small or pruning applies |
| `O(2^n)` / `O(n!)` | Enumerate subsets / permutations | Usually needs small `n`, pruning, or a stronger formulation |

When analyzing code:

- Count loop iterations, recursive calls, and the work inside each call.
- Nested loops are not automatically quadratic: inspect whether their total iterations are bounded by `O(n)` (as with two pointers that only move forward).
- State whether space is auxiliary or includes the output; mention recursion-stack depth.
- Treat hash operations as expected `O(1)`, not guaranteed worst-case `O(1)`.
- Distinguish amortized costs (for example, dynamic-array append) from per-operation worst cases.
- Include key-domain assumptions. For example, a counting array may be faster than a map but requires a small bounded range.

Constraints are evidence. If `n` is around `10^5`, a full quadratic scan is usually a warning; if the search space is small, exhaustive search may be the clearest correct solution. These are heuristics, not universal cutoffs: language, constants, input distribution, and the exact operation count still matter.

## Core structures and the operations they support

Costs below assume common implementations. “Expected” and “amortized” are intentional qualifications.

| Structure | Strong operations | Typical costs | Interview use and limitation |
| --- | --- | --- | --- |
| Array / dynamic array | Index, scan, append | Index `O(1)`; append amortized `O(1)`; middle insert/delete `O(n)` | Contiguous sequences, sorting, two pointers, windows; insertion shifts elements |
| String | Index, scan, substring checks | Index `O(1)` for common string models; constructing slices may copy | Often immutable; use an output buffer for repeated construction |
| Hash map / set | Lookup, insert, membership, frequency | Expected `O(1)` per operation; `O(n)` space | Complements, deduplication, counts, visited states; no sorted order |
| Stack | Push / pop at one end | `O(1)` | Nested structure, undo, monotonic scans, iterative DFS |
| Queue / deque | Add/remove at ends | `O(1)` | BFS, sliding-window extrema; use a deque rather than shifting an array |
| Linked list | Insert/delete with node reference | `O(1)` local relinking; locating a node `O(n)` | Pointer manipulation and cycle problems; poor random access |
| Heap / priority queue | Read/remove min or max | Peek `O(1)`; insert/remove `O(log n)`; build heap `O(n)` | Repeatedly take the next best item, top `k`, merge sorted streams; arbitrary search is `O(n)` |
| Balanced search tree | Ordered lookup, predecessor/successor | `O(log n)` | Dynamic ordered sets and ranges; less often hand-implemented in interviews |
| Trie | Follow a key/prefix | `O(L)` for key length `L` | Prefix search over strings; memory can be large |
| Disjoint-set union (DSU) | Find and union components | Amortized near-constant `O(alpha(n))` with path compression and union by size/rank | Connectivity under repeated unions; does not provide path details |
| Graph adjacency list | Visit neighbors | `O(V + E)` full traversal | Sparse graphs; memory `O(V + E)` |
| Graph adjacency matrix | Test edge existence | `O(1)` test; `O(V^2)` space and traversal scan | Dense/small graphs; often wasteful for sparse graphs |

`L` denotes key length, `V` vertices, `E` edges, and `alpha` the inverse Ackermann function, which grows so slowly that it is at most a small constant in practical input ranges.

## Choosing a representation

Ask what operation needs to be fast:

- Need to ask “have I seen this?” -> set.
- Need counts or index-by-value -> map.
- Need the smallest/largest item repeatedly -> heap.
- Need the next greater/smaller item while scanning -> monotonic stack.
- Need first-in-first-out exploration or shortest path by edge count -> queue.
- Need sorted traversal or predecessor/successor under updates -> ordered set/map.
- Need prefixes of many words -> trie.
- Need connectivity after merging groups -> DSU.
- Need random access and a one-pass scan -> array.

A structure is not an algorithm by itself. A hash map can support a one-pass complement search, prefix-sum counting, or graph visitation; the task's invariant determines which use is correct.

## Recursion, graphs, and complexity

Recursion is a way to express a problem in terms of smaller subproblems. A correct recursive design needs:

1. A precise meaning for the function.
2. A base case that terminates.
3. A recursive step that makes progress toward the base case.
4. A proof that combining the smaller answers solves the current instance.

For a tree, recursion often follows naturally because each child is a smaller tree. For a graph, cycles mean a visited set is usually essential. Recursion depth can be `O(n)` on a skewed tree or long path; use an explicit stack if that depth may exceed the language runtime limit.

## A practical selection rule

Write the required operations before selecting a data structure. Then compare:

1. Which operation dominates runtime?
2. Does the input already have useful order or structure?
3. What extra memory is acceptable?
4. Are the cost guarantees worst-case, expected, or amortized?
5. Does the representation preserve information the output requires?

Continue to the [pattern catalog](pattern-catalog.md) to connect those operations to common algorithmic approaches, or use the [problem-solving framework](problem-solving-framework.md) to work through a problem in interview order.

## Further reading

- [Princeton Algorithms: fundamentals](https://algs4.cs.princeton.edu/10fundamentals/)
- [Princeton Algorithms: sorting](https://algs4.cs.princeton.edu/20sorting/)
- [Princeton Algorithms: searching](https://algs4.cs.princeton.edu/30searching/)
- [Tech Interview Handbook: DSA study cheatsheets](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/)
