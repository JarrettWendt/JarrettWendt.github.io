---
title: Porting to Linux
author: Jarrett Wendt
excerpt: '&nbsp;'
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
thumb: assets/images/LinuxLogo.png
layout: post
cwd: '../'
---

<figure>
<img src="{{ page.cwd }}{{ page.thumb }}" alt="WargamesTicTacToe" width="500" style="display: block; margin-left: auto; margin-right: auto;">
<figcaption>Linux Logo by <a href="https://www.reddit.com/r/linux/comments/gwegv5/linux_doesnt_have_a_logo_heres_how_id_do_it/" target="_blank">u/TypicalCitrus</a></figcaption>
</figure>

Up until now, my engine has only been working on Windows. However I don't have any major Windows-only dependencies such as DirectX. The only reasons it's been Windows-only up until now is because I've been using Microsoft-specific tools, namely Visual Studio, MSBuild, MSVC, and Microsoft's C++ Unit Testing Framework. Supporting Linux would at the very least require a compiler switch, which is a tall order considering how heavily my engine relies on still experimental C++20 features.

Luckily, <a href="https://en.cppreference.com/w/cpp/compiler_support" target="_blank">the three major compilers have nearly the same level of C++20 features implemented</a>. So now I have to choose which compiler to use for Linux: GCC or Clang. If I'm using Clang for Linux, I might as well also use it for Windows since it's a cross-platform compiler. So on Windows the version of Clang I'm using is clang-cl which is what you get through the Visual Studio Installer. On Linux I'm using the latest stable release of Clang which is version 10.

I don't currently have any Linux machines for me to run my code on. Besides, it wouldn't be a great workflow if I had to switch to a completely different computer for working on a different version of my engine, or if I had to reboot my computer into Linux just to do the same. Luckily, Visual Studio works great with <a href="https://docs.microsoft.com/en-us/windows/wsl/" target="_blank">WSL</a> for being able to switch between platforms with a mere toggle in the build system.

Speaking of the build system, I unfortunately won't be able to stick with MSBuild. The Microsoft Build Engine is what everyone is using by default in Visual Studio. Whenever you go into project properties and mess with your compiler settings, that's you configuring MSBuild. You can also manually edit the .uproject files which are really XML files, but I rarely have a need to. MSBuild can be used to build and run to Linux through either a remote connection or locally through WSL. Unfortunately, when targeting Linux the latest C++ version available is C++17. Everything else about this system works great, except the lack of a C++20 or C++latest flag completely killed my ability to move forward with MSBuild.

As far as I can tell, that leaves me with CMake as the only cross-platform build system I can use. CMake is very different from MSBuild. Rather than using Visual Studio's GUI for configuring builds, you write code in CMakeLists.txt files. CMake uses its own strange language which I'm not even sure I would call C-like. It has macros, functions, curly braces, and if-statements similar to C, but is very dissimilar from C in other syntactical aspects such as assignment expressions. Typically, you set up a hierarchy of CMakeLists.txt files where you have one in each directory and CMakeLists.txt files in parent directories invoke the CMakeLists.txt files in their child directories.

Visual Studio has great support for CMake projects. You can open any folder containing a CMakeLists.txt file and it will decompose the project into your Solution Explorer. It'll re-run the CMake files whenever you edit one, and of course you can compile, run, and debug your projects. Visual Studio also provides a pseudo-replacement for the Project Properties GUI through the use of CMakeSettings.json files which can be used to edit some common things usually done in CMake such as compiler and linker flags.

---

With hours of work already put into merely discovering and setting up a build system, I can finally start refactoring my C++ code to let it compile cross-platform in Clang. There was a lot less work here than I'd expected there would be. Here's a brief summary of the types of changes I had to make:
- Clang doesn't seem to like the use of `[[nodiscard]]` on `operator`s, so I had to remove a bunch.
- MSVC-specific `#pragma warning(disable: _)`s had to be replaced with `#pragma clang diagnostic ignore ""`
- Some Windows-specific files and their usage such as `crtdbg.h` and of course `Windows.h` had to be wrapped in `#ifdef _WIN32`.
- I had to create some custom Concepts of my own to replace Microsoft extensions such as `std::_Is_rand_iter`.

I was very worried I'd have to make a lot more changes since I'm using a lot of complex templated metaprogramming in this library which I'm paranoid will be fragile and easily break. Instead, the biggest problem was merely my coroutines library. Both of the mainstream standard library implementations available to Clang, libc++ and libstdc++, have incomplete coroutine support as of this writing on both Windows and Linux. So unless I want to write those headers myself and contribute to the open source LLVM project, I have no choice to temporarily disable coroutine support in my engine until those libraries are updated. This is a painful loss since I'm very proud of how syntactically clean my coroutine system came out. At the same time, nothing else in the engine depends on them, so if anything were to completely break I'm glad it was just coroutines.

I also of course had to install Python and their debug libraries in WSL in order for my Python bindings to work cross-platform. Feeding the include and library files to the compiler and linker aside, I had to make no other changes for this aspect of my engine. It's fair to say that Python bindings "just work" cross-platform and cross-compiler.

---

Next to setting up the build system in the first place, the most time consuming part of supporting Linux was to port all of my unit tests. I have tens of thousands of lines of unit test code covering the entire codespace of my engine. I had been previously using Microsoft's C++ Unit Test framework because it comes with Visual Studio and it of course integrates with Visual Studio's excellent Test Explorer. However, it's not available in Linux. It's absolutely imperative that I be able to unit test on every platform I support. It's not sufficient to only unit test on Windows and ~~trust~~ hope that it works on Linux too.

So I had to switch to a different unit testing framework called <a href="https://github.com/catchorg/Catch2" target="_blank">Catch2</a>. This is an excellent, open source, cross-platform, cross-compiler, header-only library that can be easily installed through vcpkg (vcpkg also works on Linux which is awesome). It was an exhausting 13 hours to convert all my unit tests from one framework to another, but very worth it. Here's a few highlights of what I prefer in Catch2 over MS C++ Unit Test:
- One universal `REQUIRE` macro instead of separate `AreEqual`, `AreNotEqual`, `AreSame`, `IsTrue`, etc. is much more convenient.
- I don't need to make sure there's a `ToString` for every single type I ever want to test.
- Catch2 has its own `TEMPLATE_TEST_CASE` which is extremely useful for testing containers such as my `Datum`. I had previously hacked together my own version of this on top of MS C++ Unit Test but Catch2 makes everything so much easier.

To be fair, here's a few things about Microsoft's framework which I prefer:
- Test Explorer is really cool. I like being able to only run or debug a specific test.
- While the `REQUIRE` macro is more convenient, something about me still likes the `Assert` static class since it uses C++ templates rather than the preprocessor.
- Stepping into a `REQUIRE` macro isn't very helpful. There's about a half dozen Catch2 things being run for every `REQUIRE` and it can be difficult to step into _your code_ that's running inside the `REQUIRE`. It's honestly easier to just put a breakpoint inside the function or method being called, but that's not very easy if you're not sure which overload or template specialization is being called.
- Unit tests in Microsoft's framework run noticeably faster. I'm not sure why Catch2 is so slow. The same tests which took 100ms to run in Microsoft's framework take 5 seconds in Catch2. At the moment this isn't a big deal, but if I'm trying to debug a specific test repeatedly or the project gets significantly bigger, these seconds can add up.

---

All in all, supporting Linux in addition to Windows wasn't that bad. In general, here's a few truths I've learned that make cross-platform support easier:
- Decide early on all the platforms you intend to support. Adding another platform later into the project is much more difficult than developing with it in mind as you iterate.
- Consider using cross-platform compilers such as Clang. In general, the least number of differences you have between platforms the better.
- Avoid the use of platform-specific libraries such as DirectX as much as possible. This is of course a tall or perhaps impossible order though if you're supporting platforms like Xbox which _require_ that you use DirectX. What would help in this specific example is to use an abstraction layer such as <a href="https://github.com/bkaradzic/bgfx" target="_blank">bgfx</a>. Of course, a game engine like I'm making is _should include_ an extraction layer like this, so ideally anyone using my engine for both Windows and Linux wouldn't suffer through a problem like this.

The side effects of this compiler switch will continue. Now that I'm using a different compiler, there's a different selection of C++20 features I can attempt to use. With Clang, it might now be possible to use `operator<=>` more widely or to organize my entire library into modules instead of header and source files. Stay tuned!
