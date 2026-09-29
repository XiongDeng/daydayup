Algorithm is correct and optimal — single pass, O(n) time, O(1) space. You also changed the signature to `const std::vector<int>&`, which LeetCode doesn't give you by default. That's the right instinct and worth noting.

**The one real issue: `minPrice = 10001` is a magic number.**

You've hardcoded a value derived from LeetCode's stated constraint (`0 <= prices[i] <= 10^4`). It works, but it silently breaks if constraints change, and it tells a reviewer that your code depends on problem metadata rather than on the data itself. Three better options:

```cpp
int minPrice = INT_MAX;              // <climits>, intent is obvious
int minPrice = numeric_limits<int>::max();   // <limits>, the C++ idiom
int minPrice = prices[0];            // seed from data — but needs an empty check
```

`INT_MAX` is the pragmatic choice under time pressure. Seeding from `prices[0]` is arguably cleanest since it can't overflow and needs no sentinel, but then you need `if (prices.empty()) return 0;` first.

**Minor points:**

The `else` branch is correct but subtly load-bearing — worth being able to explain _why_. When `prices[i] < minPrice`, the profit would be negative, so skipping it is safe. Some people write it without the else:

```cpp
for (int p : prices) {
    minPrice = min(minPrice, p);
    maxProfit = max(maxProfit, p - minPrice);
}
```

That's shorter and still correct (after updating `minPrice`, `p - minPrice` is 0 on the day you buy, never negative). Either version is fine — but the range-for reads more idiomatically, and `min`/`max` from `<algorithm>` is vocabulary worth using.

**The `(int)` cast is a C-style cast. `static_cast<int>` is the C++ idiom, though with the range-for you avoid the index entirely.**

Variable shadowing: `maxProfit` the local shadows `maxProfit` the function name. Legal, works fine, but a reviewer might flag it. `best` or `result` avoids the question.

---

**Interview questions to expect:**

_"Complexity?"_ — O(n) time, O(1) space. Say it unprompted.

_"Why is one pass enough? Why don't you need to check every buy/sell pair?"_ — Because for any sell day, the best buy is the minimum price seen _before_ it. You're computing that minimum incrementally. This is the insight they're testing.

_"What if you could do unlimited transactions?"_ — LC 122. Sum every positive daily delta: `if (prices[i] > prices[i-1]) profit += prices[i] - prices[i-1];`. Still O(n).

_"At most two transactions?"_ — LC 123, genuinely harder. Track four states: best profit after first buy, first sell, second buy, second sell. This is the DP pivot.

_"At most k transactions?"_ — LC 188, the general DP. Be ready to say "O(nk) DP with a state array" even if you don't code it.

_"What if you also need to return the buy and sell days, not just the profit?"_ — Track the index when you update `minPrice`, and a second pair of indices when you update `maxProfit`. Careful: you can't just record the current `minPrice` index at sell time without having saved it.

_"What if prices arrive as a stream and you can't store them?"_ — Your algorithm already handles this. It only needs the current element and two running values. That's a nice answer, and it connects directly to your real work — this is exactly a streaming aggregation over telemetry.

_"Why did you initialize `minPrice` to 10001?"_ — Have the honest answer ready: "from the constraints; `INT_MAX` would be more robust." Owning that beats defending it.

_"Can profit be negative?"_ — No, because you may decline to trade. That's why `maxProfit` starts at 0 rather than `INT_MIN`. Worth being explicit about, since it's a deliberate design choice, not an accident.