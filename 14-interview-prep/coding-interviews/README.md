# Coding Interview Preparation: Data Structures and Algorithms

[Back to interview preparation](../README.md) · [Data structures and algorithms](../../03-data-structures-algorithms/README.md)

This is a theory-grounded, practical guide to solving coding interview problems. The goal is not to memorize a list of solutions. It is to recognize the structure of an unfamiliar problem, choose a fitting model and algorithm, justify why it works, and communicate the trade-offs clearly.

No roadmap can guarantee an interview outcome. Use this guide to build observable, repeatable skills and adapt the pace to your starting point, available time, and target role.

## Start here

1. Read [Foundations and data structures](foundations.md) if you need to refresh complexity or basic operations.
2. Learn the [problem-solving framework](problem-solving-framework.md) before practicing under time pressure.
3. Use the [pattern catalog](pattern-catalog.md) as a set of hypotheses to test against problem properties, not as a keyword lookup table.
4. Work through [dynamic programming](dynamic-programming.md) after recursion, trees, and basic recurrence relations feel familiar.
5. Follow the [12-week roadmap](roadmap.md), or use its shorter-plan guidance if your interview is sooner.

## What proficiency looks like

You are becoming interview-ready when you can consistently:

- Restate the task, clarify assumptions, and derive useful examples before coding.
- Produce a correct baseline and estimate its time and space costs.
- Use constraints and required operations to narrow the solution space.
- Explain why a pattern fits and state the invariant, recurrence, or proof idea that makes it correct.
- Implement the approach in a familiar language without relying on fragile memorized code.
- Test boundary cases, analyze complexity, and discuss a meaningful alternative or trade-off.
- Solve a mix of familiar and unseen problems while explaining your reasoning aloud.

These are better readiness signals than the number of questions completed.

## How to use pattern practice

For each new technique:

1. Learn the data structure or algorithm well enough to explain its operations.
2. Study one example and identify the property that makes the technique applicable.
3. Solve a small number of representative problems without looking at a solution.
4. After a miss, write down the specific gap: modeling, pattern selection, invariant, implementation, or testing.
5. Re-solve the problem later from a blank editor, then try a variant that changes an assumption.

When stuck, do not guess a pattern from a keyword alone. Ask: what is being selected or maintained; what can be ruled out; what repeated work can be reused; and what does the input size permit?

## Guide map

| Guide | What it covers |
| --- | --- |
| [Foundations and data structures](foundations.md) | Complexity, core structures, operation costs, and choosing a representation |
| [Problem-solving framework](problem-solving-framework.md) | A repeatable interview loop, pattern selection, proof ideas, and common confusions |
| [Pattern catalog](pattern-catalog.md) | Common DSA patterns, applicability conditions, invariants, and starting problems |
| [Dynamic programming](dynamic-programming.md) | State design, recurrence, base cases, evaluation order, and major DP families |
| [12-week roadmap](roadmap.md) | Topic sequence, weekly outcomes, deliberate practice, and readiness checks |

## A compact interview loop

**Clarify -> Model -> Baseline -> Improve -> Justify -> Implement -> Verify**

Before coding, agree on input assumptions and examples. During problem solving, explain a simple correct approach before optimizing it. Before finishing, state complexity and test normal, boundary, and adversarial cases. See the [full framework](problem-solving-framework.md) for prompts and examples.

## Source notes and further study

The guide synthesizes the following resources in original language; use them for additional problem sets and alternate explanations:

- [AlgoMaster: 15 LeetCode Patterns](https://blog.algomaster.io/p/15-leetcode-patterns) - examples spanning prefix sums, pointers, windows, stacks, heaps, and intervals.
- [AlgoMaster: 20 Patterns to Master Dynamic Programming](https://blog.algomaster.io/p/20-patterns-to-master-dynamic-programming) - a broad taxonomy of DP problem shapes and practice links.
- [Tech Interview Handbook: Coding Interview Preparation](https://www.techinterviewhandbook.org/coding-interview-prep/) - preparation strategy, communication, practice, and mock interviews.
- [Tech Interview Handbook: Coding Interview Study Plan](https://www.techinterviewhandbook.org/coding-interview-study-plan/) - a topic-based plan and advice for balancing breadth and depth.
- [Tech Interview Handbook: Coding Interview Techniques](https://www.techinterviewhandbook.org/coding-interview-techniques/) - ways to model, break down, and optimize a problem.
- [Tech Interview Handbook: Coding Interview Cheatsheet](https://www.techinterviewhandbook.org/coding-interview-cheatsheet/) - interview behaviors, from clarification through testing.
- [Tech Interview Handbook: DSA Study Cheatsheets](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/) - topic guides and a broader DSA topic inventory.
- [Grind 75](https://www.techinterviewhandbook.org/grind75/) - a customizable practice-question planner.
- [LeetCode Top Interview 150](https://leetcode.com/studyplan/top-interview-150/) - a broad curated problem set; use selectively rather than treating completion as a guarantee.
- [Princeton Algorithms, 4th Edition site](https://algs4.cs.princeton.edu/home/) - deeper foundations, lectures, exercises, sorting, searching, graphs, and strings.

The handbook's preparation, study-plan, technique, interview-practice, and DSA-cheatsheet pages and both AlgoMaster articles were reviewed on 2026-10-30. Its linked individual DSA topic pages could not be retrieved during research, so this guide uses the accessible cheatsheet index for topic coverage and does not rely on those page-specific explanations.
