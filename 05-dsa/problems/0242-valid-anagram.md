# 242. Valid Anagram

[LeetCode](https://leetcode.com/problems/valid-anagram/) · Pattern: arrays and hashing · Solved: 2026-10-19 · Revisit: 2026-10-26 · [Back to log](../problem-log.md)

## Problem
Given strings `s` and `t`, return true if `t` uses exactly the same letters as `s`, the same number of times. LeetCode limits: lowercase English letters only, length 1 to 5×10⁴.

## Approach
Frequency count in one `int[26]`: +1 for each letter of `s`, −1 for each letter of `t`. If every slot ends at 0, the counts match.

## Edge cases tested
| Input | Output | Why |
| --- | --- | --- |
| `"anagram"`, `"nagaram"` | true | Every count returns to 0 |
| `"aab"`, `"abb"` | false | a ends at +1, b at −1. A `Set` would wrongly say true |
| `"ab"`, `"a"` | false | b stays at 1. Correct even without a length check |
| `""`, `""` | true | Nothing counted, all zeros |
| `"a"`, `"A"` | crash | `'A' - 'a'` = −32 → `ArrayIndexOutOfBoundsException`. Outside LeetCode's limits, but say the assumption |

## Complexity
- Time: O(n), one pass over each string plus a fixed 26-slot check.
- Space: O(1), the array is always 26 ints, whatever the input size.

## Mistakes
- No early `s.length() != t.length()` check (still correct, just slower to say no).
- Assumed lowercase without saying it. For Unicode, use `HashMap<Character, Integer>`.
- Helper overloaded as `isAnagram(int[])`. Rename it to `allZero(int[])` for clarity.

## Code
```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if (s == null && t == null) return true;
        if (s == null || t == null) return false;
        if (s.length() != t.length()) return false;   // added after review
        int[] count = new int[26];
        for (int i = 0; i < s.length(); i++) {
            count[s.charAt(i) - 'a']++;
            count[t.charAt(i) - 'a']--;
        }
        return allZero(count);
    }

    private boolean allZero(int[] count) {
        for (int c : count) if (c != 0) return false;
        return true;
    }
}
```
