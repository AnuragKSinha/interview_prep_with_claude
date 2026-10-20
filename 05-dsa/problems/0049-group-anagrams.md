# 49. Group Anagrams

[LeetCode](https://leetcode.com/problems/group-anagrams/) · Pattern: arrays and hashing · Solved: 2026-10-20 · Revisit: 2026-10-23 · [Back to log](../problem-log.md)

## Problem
Given an array of strings, group the words that are anagrams of each other. Return the groups in any order. LeetCode limits: 1 to 10⁴ words, each 0 to 100 lowercase English letters.

## Approach
Two words are anagrams when they have the same letter counts. So give each word a "signature" key built from its counts, and group the words in a `HashMap<key, List<String>>`.

1. For each word, count its letters in an `int[26]`.
2. Turn the counts into a key: each count with `#` in front of it, e.g. `#1#0#0…`. The `#` matters: without it, a=1, b=11 and a=11, b=1 would both become `111…`.
3. `computeIfAbsent(key, k -> new ArrayList<>()).add(word)`.
4. Return `new ArrayList<>(map.values())`.

Alternative key: sort the word's letters (`"eat"` → `"aet"`). It's simpler to write, but costs O(n·m log m) in total.

## Edge cases tested
| Input | Output | Why |
| --- | --- | --- |
| `["eat","tea","tan","ate","nat","bat"]` | `[["eat","tea","ate"],["tan","nat"],["bat"]]` | Same counts → same key → same list |
| `[""]` | `[[""]]` | All-zero key; an empty string is still a group |
| `["a"]` | `[["a"]]` | One word, one group |
| `["abc","def"]` | `[["abc"],["def"]]` | No anagrams, so every key is different |
| `["a","a"]` | `[["a","a"]]` | Duplicates share a key and both stay in the list |
| `[]` | `[]` | Outside LeetCode's limits (length ≥ 1). The guard is defensive, not required |

## Complexity
- Time: O(n·(m + 26)) = O(n·m), where n = number of words and m = average length. Counting is O(m) per word; building the key is a fixed 26 steps.
- Space: O(n·m), for the map keys and the grouped words.

## Mistakes
- Left `System.out.println(key)` in. Remove debug output before submitting, because it adds noise and slows the run.
- Used the raw type `new HashMap()`. Write `new HashMap<>()`.
- Used `if/else` with `get` + `put` again. Use `computeIfAbsent(...).add(s)` instead; no second `put`, because the list is changed in place.
- Copied the groups out with a loop over `entrySet()`. Use `new ArrayList<>(ans.values())` instead.
- Assumed lowercase a–z (fine for LeetCode). For Unicode, use a `HashMap<Character, Integer>` count or the sorted key.

## Code
```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        if (strs == null || strs.length == 0)
            return new ArrayList<>();
        Map<String, List<String>> ans = new HashMap<>();
        for (String s : strs) {
            int[] ch = new int[26];
            for (char c : s.toCharArray()) {
                ch[c - 'a']++;
            }
            String key = stringify(ch);
            ans.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
        }
        return new ArrayList<>(ans.values());
    }

    private String stringify(int[] ch) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 26; i++) {
            sb.append('#').append(ch[i]);
        }
        return sb.toString();
    }
}
```
