---
permalink: /sandbox
title: Sandboxing
---

Luau is safe to embed. Broadly speaking, this means that even in the face of untrusted (and in Roblox case, actively malicious) code, the language and the standard library don't allow any unsafe access to the underlying system, and don't have any bugs that allow escaping out of the sandbox (e.g. to gain native code execution through ROP gadgets et al). Additionally, the VM provides extra features to implement isolation of privileged code from unprivileged code and protect one from the other; this is important if the embedding environment (Roblox) decides to expose some APIs that may not be safe to call from untrusted code, for example because they do provide controlled access to the underlying system or risk PII exposure through fingerprinting etc.

This safety is achieved through a combination of removing features from the standard library that are unsafe, adding features to the VM that make it possible to implement sandboxing and isolation, and making sure the implementation is safe from memory safety issues using fuzzing.

Of course, since the entire stack is implemented in C++, the sandboxing isn't formally proven - in theory, compiler or the standard library can have exploitable vulnerabilities. In practice these are usually found and fixed quickly. While implementing the stack in a safer language such as Rust would make it easier to provide these guarantees, to our knowledge (based on existing code) this would make it impossible to reach the level of performance required.

## Library

Parts of the Lua 5.x standard library are unsafe. Some of the functions provide access to the host operating system, including process execution and file reads. Some functions lack sufficient memory safety checks. Some functions are safe if all code is untrusted, but can break the isolation barrier between trusted and untrusted code.

The following libraries and global functions have been removed as a result:

- `io.` library has been removed entirely, as it gives access to files and allows running processes
- `package.` library has been removed entirely, as it gives access to files and allows loading native modules
- `os.` library has been cleaned up from file and environment access functions (`execute`, `exit`, etc.). The only supported functions in the library are `clock`, `date`, `difftime` and `time`.
- `debug.` library has been removed to a large extent, as it has functions that aren't memory safe and other functions break isolation; the only supported functions are `traceback` ~~and `getinfo` (with reduced functionality)~~.
- `dofile` and `loadfile` allowed access to file system and have been removed.

To achieve memory safety, access to function bytecode has been removed. Bytecode is hard to validate and using untrusted bytecode may lead to exploits. Thus, `loadstring` doesn't work with bytecode inputs, and `string.dump`/`load` have been removed as they aren't necessary anymore. When embedding Luau, bytecode should be encrypted/signed to prevent MITM attacks as well, as the VM assumes that the bytecode was generated by the Luau compiler (which never produces invalid/unsafe bytecode).

Finally, to make isolation possible within the same VM, the following global functions have reduced functionality:

- `collectgarbage` only works with `"count"` argument, as modifying the state of GC can interfere with the expectations of other code running in the process. As such, `collectgarbage()` became an inferior version of `gcinfo()` and is deprecated.
- `newproxy` only works with `true`/`false`/`nil` arguments.
- `module` allowed overriding global packages and was removed as a result.

> Note: `getfenv`/`setfenv` result in additional isolation challenges, as they allow injecting globals into scripts on the call stack. Ideally, these should be disabled as well, but unfortunately Roblox community relies on these for various reasons. This can be mitigated by limiting interaction between trusted and untrusted code, and/or using separate VMs.

## Environment

The modification to the library functions are sufficient to make embedding safe, but aren't sufficient to provide isolation within the same VM. It should be noted that to achieve guaranteed isolation, it's advisable to load trusted and untrusted code into separate VMs; however, even within the same VM Luau provides additional safety features to make isolation cheaper.

When initializing the default globals table, the tables are protected from modification:

- All libraries (`string`, `math`, etc.) are marked as readonly
- The string metatable is marked as readonly
- The global table itself is marked as readonly

This is using the VM feature that is not accessible from scripts, that prevents all writes to the table, including assignments, `rawset` and `setmetatable`. This makes sure that globals can't be monkey-patched in place, and can only be substituted through `setfenv`.

By itself this would mean that code that runs in Luau can't use globals at all, since assigning globals would fail. While this is feasible, in Roblox we solve this by creating a new global table for each script, that uses `__index` to point to the builtin global table. This safely sandboxes the builtin globals while still allowing writing globals from each script. This also means that short of exposing special shared globals from the host, all scripts are isolated from each other.

## Thread identity

Environment-level sandboxing is sufficient to implement separation between trusted code and untrusted code, assuming that `getfenv`/`setfenv` are either unavailable (removed from the globals), or that trusted code never interfaces with untrusted code (which prevents untrusted code from ever getting access to trusted functions). When running trusted code, it's possible to inject extra globals from the host into that global table, providing access to special APIs.

However, in some cases it's desirable to restrict access to functions that are exposed both to trusted and untrusted code. For example, both may have access to `game` global, but `game` may expose methods that should only work from trusted code.

To achieve this, each thread in Luau has a security identity, which can only be set by the host. Newly created threads inherit identities from the parent thread, and functions exposed from the host can validate the identity of the calling thread. This makes it possible to provide APIs to trusted code while limiting the access from untrusted code.

> Note: to achieve an even stronger guarantee of isolation between trusted and untrusted code, it's possible to run it in different Luau VMs, which is what Roblox does for extra safety.

## `__gc`

Lua 5.1 exposes a `__gc` metamethod for userdata, which can be used on proxies (`newproxy`) to hook into garbage collector. Later versions of Lua extend this mechanism to work on tables.

This mechanism is bad for performance, memory safety and isolation:

- In Lua 5.1, `__gc` support requires traversing userdata lists redundantly during garbage collection to filter out finalizable objects
- In later versions of Lua, userdata that implement `__gc` are split into separate lists; however, finalization prolongs the lifetime of the finalized objects which results in less prompt memory reclamation, and two-step destruction results in extra cache misses for userdata
- `__gc` runs during garbage collection in context of an arbitrary thread which makes the thread identity mechanism described above invalid
- Objects can be removed from weak tables *after* being finalized, which means that accessing these objects can result in memory safety bugs, unless all exposed userdata methods guard against use-after-gc.
- If `__gc` method ever leaks to scripts, they can call it directly on an object and use any method exposed by that object after that. This means that `__gc` and all other exposed methods must support memory safety when called on a destroyed object.

Because of these issues, Luau does not support `__gc`. Instead it uses tag-based destructors that can perform additional memory cleanup during userdata destruction; crucially, these are only available to the host (so they can never be invoked manually), and they run right before freeing the userdata memory block which is both optimal for performance, and guaranteed to be memory safe.

For monitoring garbage collector behavior the recommendation is to use weak tables instead.

## Interrupts

In addition to preventing API access, it can be important for isolation to limit the memory and CPU usage of code that runs inside the VM.

By default, no memory limits are imposed on the running code, so it's possible to exhaust the address space of the host; this is easy to configure from the host for Luau allocations, but of course with a rich API surface exposed by the host it's hard to eliminate this as a possibility. Memory exhaustion doesn't result in memory safety issues or any particular risk to the system that's running the host process, other than the host process getting terminated by the OS.

Limiting CPU usage can be equally challenging with a rich API. However, Luau does provide a VM-level feature to try to contain runaway scripts which makes it possible to terminate any script externally. This works through a global interrupt mechanism, where the host can setup an interrupt handler at any point, and any Luau code is guaranteed to call this handler "eventually" (in practice this can happen at any function call or at any loop iteration). This still leaves the possibility of a very long running script open if the script manages to find a way to call a single C function that takes a lot of time, but short of that the interruption is very prompt.

Roblox sets up the interrupt handler using a watchdog that:

- Limits the runtime of any script in Studio to 10 seconds (configurable through Studio settings)
- Upon client shutdown, interrupts execution of every running script 1 second after shutdown