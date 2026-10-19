# 1. Two Sum

[LeetCode](https://leetcode.com/problems/two-sum/) · Pattern: arrays and hashing · Solved: 2026-10-12 · Revisit: 2026-10-23 · [Back to log](../problem-log.md)

## Problem
Given `nums` and `target`, return the indices of the two numbers that add up to `target`. Exactly one answer exists, and an element can't be used twice. Values and target are in [-10⁹, 10⁹].

## Approach
One pass with a `HashMap<Integer, Integer>` (value → index). For each `nums[i]`, look up `target - nums[i]`. If it's there, return both indices. If not, `put(nums[i], i)`. Checking **before** putting means an element never pairs with itself.

## Edge cases tested
| Input | Output | Why |
| --- | --- | --- |
| `[3,3]`, 6 | `[0,1]` | First 3 is stored, second 3 finds it |
| `[3,2,4]`, 6 | `[1,2]` | Check-before-put stops 3 pairing with itself |
| `[-3,4,3,90]`, 0 | `[0,2]` | Negatives work the same way |
| `[2,2,5]`, 7 | `[1,2]` | The second `put(2, 1)` overwrites, but same value, so no answer is lost |

## Complexity
- Time: O(n), one pass, and each HashMap get/put is O(1) on average. Since Java 8 a crowded bucket becomes a red-black tree, so the worst case is O(log n) per lookup.
- Space: O(n), the map holds up to n−1 entries.

## Mistakes
- Overflow: max `target - nums[i]` = 2×10⁹, which is under `Integer.MAX_VALUE`, so it fits. Int overflow wraps silently, so check the limits every time.
- Gave edge-case outputs without the reason.
- `containsKey()` then `get()` = two lookups. Use one `get()` and a null check.
- Full detail: [10-mistakes.md](../../10-mistakes.md) (2026-10-12).

## Code
```java
// paste your solution here on the revisit day (re-solve it cold first)
```
