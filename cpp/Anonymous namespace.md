The inner `namespace { ... }` with no name is an **anonymous namespace**. It makes `close_fd` private to that one file, `src/util.cpp`.

**What it does**

- Everything inside an anonymous namespace has _internal linkage_. The name is visible only in the same `.cpp` file, and other files can't see or call it.
- `namespace wws` is the project's public API, declared in the headers: `run_command`, `trim`, `log`, and so on. Other files can call those.
- `close_fd` is a small helper used only inside `run_command`. I don't want it to be part of the public interface.

**Why not just a plain function in `wws`?**

- A plain `wws::close_fd` has external linkage. If another `.cpp` file later defined its own `wws::close_fd`, the linker would report a duplicate symbol, or the two could clash.
- With the anonymous namespace, each file can have its own `close_fd` with no clash.
- The compiler can also warn if a helper like that is never used, and it may inline it more freely.

**`static` instead**  
`static void close_fd(...)` does the same job. C++ prefers anonymous namespaces for functions, and `static` is the older C habit. Other files in this project use `static` for small helpers, such as `static std::string curl_quote` in `slack.cpp`. Both are fine. Using both styles is a small inconsistency in the code, and I can switch them all to anonymous namespaces if you'd like.

**The `::close` inside it**  
`::close(fd)` has a leading `::`, which means "the global `close` from `<unistd.h>`", not anything in `wws`. The helper is also named `close_fd`, not `close`, so it wouldn't hide that function by accident.