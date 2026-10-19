# DSA problem log

One row per problem, newest first. Re-solve each one cold on its revisit date. Solved once is not learned.
Full notes for each problem (approach, edge cases, complexity, code) are in [`problems/`](problems/), one file per problem, made from the [template](problem-template.md). Run the [edge-case checklist](edge-case-checklist.md) before calling any solution done.

| Date | Problem | Pattern | Problem in one line | Key insight in one line | Mistake I made | Revisit on | Details |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-10-19 | [242. Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Arrays and hashing | Do two strings use exactly the same letters, the same number of times? | +1 for each letter of `s`, −1 for each letter of `t` in one `int[26]`; all zeros = anagram | Crashes on non-lowercase input (`'A' - 'a'` = −32); no early length check | 2026-10-26 | [notes](problems/0242-valid-anagram.md) |
| 2026-10-12 | [1. Two Sum](https://leetcode.com/problems/two-sum/) | Arrays and hashing | Return the indices of the two numbers that add up to `target` | Map each value → its index; check for `target - nums[i]` before `put`, so nothing pairs with itself | Didn't check int overflow; gave edge-case outputs without the "why" | 2026-10-23 | [notes](problems/0001-two-sum.md) |
