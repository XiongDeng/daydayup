
Here are the ones that actually come up, roughly in order of how often you'll reach for them.

**`operator[]`** — you're already using it. Worth naming explicitly because of the insert-on-access behavior: `m[k]` default-constructs a value if `k` isn't present. That's what makes `freq[x]++` work (int default-constructs to 0), and it's also the trap that makes `if (m[k] == 0)` silently grow your map.

**`erase`** — three forms:

```cpp
m.erase(key);        // by key, returns how many removed (0 or 1)
m.erase(it);         // by iterator, returns iterator to next
m.erase(it1, it2);   // range
```

The iterator form matters when erasing during iteration:

```cpp
for (auto it = m.begin(); it != m.end(); ) {
    if (shouldRemove(it->first)) it = m.erase(it);
    else ++it;
}
```

Erasing by key inside a range-for invalidates your iterator. Classic bug.

**`emplace`** — constructs the value in place rather than constructing then copying:

```cpp
m.emplace(key, value);
```

Marginal for `int`, meaningful for expensive value types. Interviewers like seeing it.

**`insert`** — returns `pair<iterator, bool>` where the bool says whether insertion happened:

```cpp
auto [it, inserted] = m.insert({key, value});
if (!inserted) { /* key already existed, it points to the existing entry */ }
```

Unlike `operator[]`, `insert` does **not** overwrite an existing value. That difference gets asked.

**`try_emplace`** (C++17) — like `insert` but doesn't construct the value at all if the key exists. The cleanest "insert if absent" primitive:

```cpp
m.try_emplace(key, expensiveValue);
```

**`contains`** (C++20) — reads better than `count`:

```cpp
if (m.contains(key)) { ... }
```

Available on LeetCode and HackerRank, both run recent GCC. Use it when you only need a yes/no.

**`empty`** — `m.empty()` over `m.size() == 0`. Trivial but idiomatic.

**`clear`** — resets to empty, keeps the bucket allocation.

**`begin` / `end`** — you have `end()`. `begin()` matters mostly for manual iteration and the erase pattern above.

**`bucket_count` / `load_factor` / `max_load_factor`** — diagnostic, not day-to-day, but you already know the concepts from the reserve discussion. Worth recognizing if an interviewer mentions them.

---

**Two patterns worth internalizing over any individual method:**

The `find`-once idiom, which you've now applied to Two Sum:

```cpp
auto it = m.find(k);
if (it != m.end()) { use(it->second); }
```

And `it->first` / `it->second` — a map iterator points at a `pair<const Key, Value>`. The `const` on the key is why you can't modify keys in place; you erase and re-insert instead.

One method conspicuously **absent** from `unordered_map`: `lower_bound`. There's no ordering, so no range queries. If you need "smallest key ≥ x," that's `map`, not `unordered_map`. That distinction shows up in problems like Time Based Key-Value Store from your Day 7 list.



The explicit type is a nested typedef inside the container:

```cpp
unordered_map<int,int>::iterator it = hash.find(complement);
```

And if the map is `const` (or you're in a `const` member function), `find()` returns a `const_iterator` instead:

```cpp
unordered_map<int,int>::const_iterator it = hash.find(complement);
```

For other containers it follows the same shape:

```cpp
vector<int>::iterator vit = find(v.begin(), v.end(), x);
set<string>::iterator sit = s.find("key");
map<string, vector<int>>::iterator mit = m.find("key");   // getting ugly
```

That last line is the argument for `auto` in a nutshell. The type is mechanical — you're just transcribing the container's declaration — but it grows unwieldy fast, and if you later change `unordered_map` to `map`, every explicit iterator declaration breaks while every `auto` one keeps compiling.

There's a subtler reason too: with explicit types it's easy to write something that _looks_ right but forces a hidden conversion. If you write `unordered_map<int,int>::iterator` on a const map, you get a compile error. If you write `map<int,int>::iterator` when the container is actually `unordered_map`, same. `auto` can't get it wrong.

**For interviews:** use `auto`. It's universal practice in modern C++, and an interviewer will find explicit iterator types slightly dated rather than impressive. The one place explicit types still earn their keep is in public API signatures and documentation, where naming the type communicates intent to the reader.

If you want to show you know what's underneath, the move is to mention it verbally — "that's an `unordered_map<int,int>::iterator`, I'm using `auto` for readability" — rather than typing it out.

One related piece of vocabulary worth having: `decltype` gives you the type of an expression, which is how you name a lambda's type (that awkward `priority_queue` comparator syntax from your Day 4 drill uses `decltype(cmp)` for exactly this reason).




**1. Double hash lookup.** You compute the hash of `target - nums[i]` twice: once in `count()`, again in `at()`. Use `find()` once and reuse the iterator:

```cpp
auto it = hash.find(complement);
if (it != hash.end()) return {it->second, (int)i};
```

This is the single most likely thing an interviewer comments on. It signals whether you think in terms of iterators or just map lookups.

**2. Narrowing conversion.** `i` is `size_t` (unsigned 64-bit); `result[0]` is `int`. That's an implicit narrowing conversion. It's harmless here, but `-Wconversion` would flag it. Either declare `int i` and cast `nums.size()`, or `static_cast<int>(i)` explicitly. Coming from C this feels pedantic, but modern C++ reviewers care about signed/unsigned discipline.

**3. Repeated subexpression.** `target - nums[i]` appears twice. Hoist it: `int complement = target - nums[i];` Clearer, and it gives you something to name when explaining your logic aloud.

**4. Construct-then-assign.** `vector<int> result(2); result[0] = ...; result[1] = ...;` allocates, value-initializes to zero, then overwrites. `return {a, b};` does it in one step.

Tightened version:

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> seen;
    for (int i = 0; i < (int)nums.size(); ++i) {
        int complement = target - nums[i];
        auto it = seen.find(complement);
        if (it != seen.end()) return {it->second, i};
        seen[nums[i]] = i;
    }
    return {};
}
```

---

**What the interviewer will likely ask:**

_"What's the time and space complexity?"_ — O(n) time, O(n) space. Say it unprompted.

_"What's the worst case for `unordered_map`?"_ — This is the follow-up that separates candidates. O(1) is _average_; with adversarial hash collisions every insert degrades to O(n), making the whole thing O(n²). `std::unordered_map` uses chaining, so it degrades rather than breaking. Bonus points for mentioning `reserve(nums.size())` to avoid rehashing.

_"Why `unordered_map` instead of `map`?"_ — `map` is a red-black tree, O(log n) per operation, and you don't need sorted order here.

_"What if the array were sorted?"_ — Two pointers from both ends, O(n) time and **O(1) space**. Better space complexity. They love this pivot.

_"Why do you check before inserting?"_ — Subtle and worth knowing: it correctly handles the case where `nums[i] * 2 == target`. If you inserted first, an element would match itself. Your ordering is right — be ready to explain _why_ it's right, not just that it works.

_"What about duplicate values?"_ — `seen[nums[i]] = i` overwrites the earlier index. Fine here since you check before inserting, but an interviewer may ask what changes if you needed _all_ pairs (then you'd store `vector<int>` of indices per key).

_"Integer overflow?"_ — Under LeetCode's constraints (|nums[i]|, |target| ≤ 10⁹), `target - nums[i]` maxes at 2×10⁹, just under `INT_MAX` (~2.147×10⁹). Safe, but tight. If constraints widened, you'd need `long long`. Volunteering this shows the C instinct — most candidates never think about it.

_"What if the array doesn't fit in memory?"_ — Scale follow-up. External sort plus two pointers, or distributed hashing by key range. Your Alibaba and Wärtsilä background makes this an easy one to shine on.


