# DSA problem log

One row per problem, newest first. Re-solve each one cold on its revisit date. Solved once is not learned.

| Date | Problem | Pattern | Key insight in one line | Mistake I made | Revisit on |
| --- | --- | --- | --- | --- | --- |
| 2026-10-12 | [1. Two Sum](https://leetcode.com/problems/two-sum/) | Arrays and hashing | Store what each element still needs (`target - nums[i]`) → its index; check before put so nothing pairs with itself | Didn't check whether `target - nums[i]` can overflow an int; gave outputs for edge cases without the "why" | 2026-10-19 |

## Edge-case checklist (run before calling any solution done)

- [ ] Duplicates (e.g. `[3,3]`)
- [ ] An element pairing with itself (e.g. `[3,2,4]`, target 6)
- [ ] Negatives and zero (e.g. `[-3,4,3,90]`, target 0)
- [ ] Smallest input allowed (length 1 or 2, empty if allowed)
- [ ] Answer at the first or last index
- [ ] Overwrites in a map: can one lose a valid answer?
- [ ] Integer overflow on `+`, `-` or `*` of two inputs: work out the max value against `Integer.MAX_VALUE` (2,147,483,647)
- [ ] State time and space complexity **with the reason**
