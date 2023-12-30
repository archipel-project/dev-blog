+++
title = "The road to the perfect plugin system"
author = "D3lta-2-1"
date = "2023-12-29"

categories = ["dev logs"]
tags = ["plugins", "core", "research"]

description = "Exploring Modding possibilities and finding the ideal solution."

[banner]
image = "plugin.jpg"
caption = "Plugins are like legos, there are modular and allow creativity"
+++

Minecraft Java Edition became very popular thanks to its modding community, and the ability to create your own server, by customizing with plugins and mods.

For example, there are more than 144.5K mods to this date on CurseForge, more than 8K plugin on Modrinth, and this number is growing every day.

<!--more-->

# What is a plugin/mod for Archipel?

Modding is a big (maybe the biggest) part of Minecraft, so we can't ignore it. Some recent Minecraft projects like Minestom
are just a mod loaders, where the base game is also considered as mod!

Following Minestom choices, we want all content of our software to be a mod/plugin, leaving only core features in the server itself.
Even Minecraft Vanilla features will be considered as a plugin. This way, you can always choose to keep only the parts you are interested in.

> _But what is a plugin?_

A plugin is a piece of code that can be loaded to modify the program behavior, without a full recompilation or modification of the server binaries.
Plugins are the solutions we provide to developers to modify and improve the game. Allowing them to add new blocks or items, imagine new dimensions
and challenge their players with creative mechanics.

# What is a good plugin system?

To meet our needs, a good plugin system would be:

- **fast**: too many mods should not slow down the server. Ideally, it would safely handle multi-threading.
- **sand-boxed**: a plugin should not makes the server crash or access data on the host system (like sensitive files).
- **easy to use**: a plugin should be easy to create and maintain (for the developer), and easy to use (for the user).
- **cross-platform**: it has to work everywhere the server works, without recompilation, like Minecraft does with Java.
  You can run the same mod independently on Windows, Linux and Mac, even on x86 and ARM.
- **stable ABI** (Application Binary Interface): this is a must-have, plugins should be able to be compiled with different compilers (or version of a compiler)
  and keep the same output without breaking the ABI.
- **symbols importation between plugins**: have a plugin is nice, but being able to interact with other plugins is even better. Plugins should be able
  to import symbols from other plugins, like [loom](https://fabricmc.net/wiki/documentation:fabric_loom)'s `modimplementation`.
  Core mods align with the philosophy of the fabric modding community. They aim to avoid bloating the server core with excessive features, reducing the
  effort required for updates between versions. Additionally, core mods can optimize memory usage by limiting the number of loaded binaries in RAM.

Obviously, it is going to be hard, if not impossible to find a solution that meets all these criteria, we might even have to make some compromises...

In the following sections, we will explore some solutions we have considered, and why we didn't choose them.

# First thoughts: Lua/wren

Like every other plugin system, we thought about using lua. It's a very popular language that is used in many games and is very easy to embed into C/C++ applications.

The 2 mosts popular implementations are [LuaJit](https://luajit.org/) and [Wren](https://wren.io/). Wren is a little slower, but solves many problems from Lua, like the lack of classes or static typings.

Both are very lightweight and easy to use, sandbox-able but, they suffer from a big problem: they don't support any kind of multi-threading. Developers are forced to working inside a single thread or
to work with multiples VMs in parallel (Not the smartest idea, right?). This means that this is not an idea solution, since user can always store data on a specific VM, but
will not be able to access it from another.

To resume:

| Criteria                      |  Note   | Summary                                                                                                                                                          |
| ----------------------------- | :-----: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          |   2/5   | Lacks multithreading.                                                                                                                                            |
| Sand-boxed                    |   Yes   | Both provides sand-boxing capabilities.                                                                                                                          |
| Easy to use                   |   3/5   | Looks easy to use, but might quickly become messy on large projects.                                                                                             |
| Cross-platform                | Runtime | Can run literally everywhere.                                                                                                                                    |
| Stable ABI                    |   Yes   | _Nothing to comment_                                                                                                                                             |
| Cross-plugins symbols imports |   3/5   | Interactions would be very tricky to implement, since Lua imports means executing it. Getting a function from another plugin would always result in a core call. |

Why did I look for alternatives? Because Lua/wren are too primitive for our use case, and we can't rely on 25k lines of code written with it just to ensure everything run smootly.
Others points like interactions with an ECS are very tricky to implement: a component is basically a static-sized and statically typed structure, and this notion doesn't exists at all in these
languages due to their dynamic typing nature.

# Seconds though: Rust ABI using dylib

Instead of using an external library, we though about just sticking with the current dynamic linking mechanism of Rust/C ABI.
Bevy uses this approach, and is very easy to use. It only requires to add a dependency in your `cargo.yoml` and use it like any other rust libraries. In addition, compiled Rust for native platform
is the fastest option in terms of execution speed, while offering multi-threading and many other advanced features from Rust.

But, we will lose some key points here:

| Criteria                      |     Note     | Summary                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------------- | :----------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          |     5/5      | [AOT (Ahead of time)](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) is the fastest way to execute code, we are talking about 2x to 10x the speed of any other alternative.                                                                                                                                                                          |
| Sand-boxed                    |      No      | A DLL get the same level of permission as the server, it can access any file, it can crash the server, it can do anything the server can do, which is not ideal.                                                                                                                                                                                               |
| Easy to use                   |     4/5      | Rust is not the easiest language to learn, but it's not the hardest either. It's very well documented, and would fit perfectly with the rest of the codebase!                                                                                                                                                                                                  |
| Cross-platform                | Compile-time | Rust code is cross-platform at compile-tme, and can, accordingly, run on almost any platform. But, binaries need to be recompiled for each platform to ensure the support.                                                                                                                                                                                     |
| Stable ABI                    |      No      | Rust ABI is not stable, changing a flag in the compiler can change the padding of a structure, and break the ABI: plugins would need to be compiled with the exact same compiler configuration.                                                                                                                                                                |
| Cross-plugins symbols imports |     2/5      | And here start our nightmare: importing symbols from other plugins isn't really a thing, the "host" plugin must expose an array of static void pointers to every desired functions, and "guest" plugins must cast the void pointer to the desired function pointer and call it. This is not ideal at all and would result in a lot of unsafe boilerplate code. |

As you can see, this is a big no since it doesn't comply at all to our needs.

# Third though: what about C ABI?

C ABI is the most stable ABI ever, it's the ABI used by both C and C++ languages. It's also the ABI used by the Rust language when using the `#[no_mangle]` attribute.
Obviously, the solution is similar to the precedent, but the C ABI solves maybe problems related to the ABI stability, like being constrained to use the same compiler, version or flags.

But, this also does come with a trade-off: even more unsafe and boilerplate code! To map everything to the C ABI, we would need to use a lot of unsafe code, and in consequence use a lot of boilerplate
code to map Rust types and functions to their C equivalents. This is far from being ideal, but it's the best we can do with C ABI. Another issue is that the notion of trait would disappear completely,
preventing us from doing any possible abstraction in our code.

| Criteria                      |     Note     | Summary                                                                                                                                                      |
| ----------------------------- | :----------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Fast                          |     5/5      | Same as the precedent.                                                                                                                                       |
| Sand-boxed                    |      No      | Same as the precedent.                                                                                                                                       |
| Easy to use                   |     3/5      | Same as the precedent, but with more boilerplate code.                                                                                                       |
| Cross-platform                | Compile-time | A bit better than the precedent, but we can still forget about using the same binary on different OS/CPU architectures.                                      |
| Stable ABI                    |     Yes      | We are using C ABI/FFI, which is perfectly stable.                                                                                                           |
| Cross-plugins symbols imports |     1/5      | Even worse than the precedent, we can't even use a trait to abstract the function pointer. "Guest" plugins wouldn't even be able to implement "Host" traits. |

Both Rust and C ABI allow to get a reference over any object passed by the core. It means we can iterate over components from our ECS without **any** copy! This is an excellent point, a allow us to write very efficient plugins.
But as shown above, this also means a nightmare for us and plugins developers to implement anything.

# Third though: WASM? Is this worth it?

WebAssembly is a new kind of bytecode, designed to be close from native performances, but way more portable. We can compare it a more minimalistic version of the Java or C# bytecode.
WASM runs in browsers when tasks requires performances where JavaScript is too slow, or in a dedicated runtime like Wasmtime or Wasmer.

WebAssembly (WASM) offers a compelling trade-off with its performance falling between native speed and Luajit (the fastest JIT language). It offers an easy way to sandbox by design,
is highly cross-platform, has a stable ABI, and allows for importing symbols from other plugins. Calling functions in WASM is very easy, at least, with primitive types...
From the moment where you want to pass complex objects between the host and the guest, things get a bit more complicated. There is no official way to do this task cleanly.
Of course, there are always hacky way to pass a pointer to the guest, and the guest can read/write to this pointer. This would provide a terrible developer experience, and would be very error prone.
Passing complex objects requires some serialization, which comes with big performance costs and boilerplate code. It also requires a lot of handmade VM memory management and means, for us,
to reimplemente the whole ECS in WasmTime VM memory, or copying each component to the VM memory during entity query, which also comes with a cost.

| Criteria                      |  Note   | Summary                                                                                                                                               |
| ----------------------------- | :-----: | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          |   2/5   | Yes, WASM is fast, but the serialization cost and handmade VM memory management is too high to even consider it.                                      |
| Sand-boxed                    |   Yes   | It is by design!                                                                                                                                      |
| Easy to use                   |   0/5   | WASM is very easy to use, but the handmade VM memory management is too tricky to implement. (I almost jumped from the Window)                         |
| Cross-platform                | Runtime | It is by design!                                                                                                                                      |
| Stable ABI                    |   Yes   | It is by design!                                                                                                                                      |
| Cross-plugins symbols imports |   2/5   | While it is one of the better solutions we have seen so far, symbols management still poses challenges, especially when dealing with complex objects. |

Projects like [Feather](https://feathermc.org/) uses this approach, but failed to implement it, because the heavy sand-boxing and handmade VM memory management.
The trade-off related to the implementation and the performance cost is too high. They also failed to implement a proper ECS in the VM memory as they were copying each component to the VM memory
during entity query, which also come with a big performance cost.

Again, this is a big no for us.

# Fourth though: speaking of bytecode, what about Java?

Java (or any alternative running on the JVM like kotlin) is one of the most popular language today, and used to create Minecraft. It provides good speeds, portability,
and the targeted community already used it

| Criteria                      |  Note   | Summary                                                                                                                                                         |
| ----------------------------- | :-----: | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          |   3/5   | Java is fast, but the OO with deep inheritance is not the best for cache coherency, and the memory management is not the best. (Java is memory-hungry)          |
| Sand-boxed                    |   No    | I didn't see any way to sandbox java, because I found big issues for my use case, I didn't dig too much                                                         |
| Easy to use                   |   4/5   | Java is very easy to use and to learn, JNI is also surprisingly easy to use                                                                                     |
| Cross-platform                | Runtime | Java is designed for that                                                                                                                                       |
| Stable ABI                    |   Yes   | provide some bytecode versions and securities for that                                                                                                          |
| Cross-plugins symbols imports |   5/5   | allow symbol resolution at runtime, which means it does allow symbols importation between plugins out of the box. This is the best solution we have seen so far |
| _(Bonus)_                     |    -    | The Minecraft community already know it!                                                                                                                        |

Java is a very good language, with a lot of interesting features, but it also come with some drawbacks. For example, the memory management, the lack of Native function pointer
(each "function container" is a polymorphic object, making a huge overhead for our project), and the lack of "inline classes", "primitives classes" or "structure",
call it the way you want, what i mean here is a in-stack data-structure with compile time size, easy to put in array, making the ECS iteration very impractical.
The [JEP402](https://openjdk.org/jeps/402) might change that.

# Final though: and Java's little brother made by Microsoft, C#?

C# is a very good language, with good community, and very nice features like structures (in stack data object) or references (with some kind of mutability control).
Like java, C# run a VM: a single binary can be run everywhere as long as .NET is available

| Criteria                      |  Note   | Summary                                                                                                                                 |
| ----------------------------- | :-----: | --------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          |   4/5   | C# is really fast, also comme with OO and deep inheritance, but you can also use struct and more functional programming.                |
| Sand-boxed                    |   No    | I see some project talking about sand_boxing C#, but I didn't dig too much.                                                             |
| Easy to use                   |  4.5/5  | C# is very easy to use and to learn, and the C# community is very active (-0.5 because I don't like PascalCase on Methods).             |
| Cross-platform                | Runtime | As outlined above, as long as .NET is available, binaries can run everywhere (Windows/Linux/Mac, x86/x64/Arm/Arm64).                    |
| Stable ABI                    |   Yes   | stable across major version.                                                                                                            |
| Cross-plugins symbols imports |   5/5   | Allow symbol resolution at runtime, which means symbols cross-importation out of the box. This is our best solution we have seen so far |

C# have "unmanaged struct" (struct with a fixed size, and no reference), which is very good for ECS and cache coherency.
Unfortunatly, C# suffers from the same issue as Java: native function pointer are hard to get, but existent. A function can be declared as `[UnmanagedCallersOnly]` and can be called from a function pointer. This is not very ergonomic, and require a lot of boilerplate code.

I didn't find any way to use references from `hostfxr`, it might be tricky to iterate over ECS without any copy.
Threading doesn't work out of the box but should be doable with some work!

HostFxr is really minimalistic, it exposes at most 10 functions. We also looked at Mono, which is used in the Unity engine, but the project is slowly dying, with an inactive community and
stuck at C# 8.0 while .NET is already at C# 12.0 !

# Conclusion

We have explored many solutions, and obviously, none of them are perfect. We will have to make some compromises to keep the most suitable to our needs.
At the time of writing this post, we are currently working on using C# as our language of choice for modding and are creating a C# library to simplify
plugins creators life.
