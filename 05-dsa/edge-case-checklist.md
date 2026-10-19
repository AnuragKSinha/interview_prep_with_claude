# Edge-case checklist

Run this before calling any solution done. For every case, say the **output and the reason**, not just the output. Back to the [problem log](problem-log.md).

- [ ] Duplicates (e.g. `[3,3]`)
- [ ] An element pairing with itself (e.g. `[3,2,4]`, target 6)
- [ ] Negatives and zero (e.g. `[-3,4,3,90]`, target 0)
- [ ] Smallest input allowed (length 1 or 2, empty if allowed)
- [ ] Answer at the first or last index
- [ ] Overwrites in a map: can one lose a valid answer?
- [ ] Integer overflow on `+`, `-` or `*` of two inputs: work out the max value against `Integer.MAX_VALUE` (2,147,483,647)
- [ ] Input outside the stated range (uppercase, Unicode, nulls): say the assumption out loud
- [ ] Inputs of different sizes (e.g. two strings of different lengths)
- [ ] State time and space complexity **with the reason**
