# cmake enable_language is a scope trap

shipped a PR to silence a cmake policy warning. CMP0194, the new one that complains about `project()` being called with languages that haven't been enabled. the warning was cosmetic. the fix broke the windows build. the fix for the fix broke a self-hosted arm runner. the third fix was a revert.

---

## what actually happened

the original warning was harmless — cmake 3.29+ wants you to pre-declare languages before `project()`. easy. just add `enable_language(ASM)` at the top. done.

except. on windows with MSVC + ninja, cmake's language detection walks PATH looking for an assembler. if you have MinGW installed (which any developer who's touched mingw-w64 once has), cmake finds the GNU assembler first, decides that's the system ASM toolchain, and tries to use it with MSVC-generated call conventions. the build explodes in a way that looks nothing like an assembler problem.

fine, scope it. the ASM dependency was actually only needed by one third-party subdirectory (kleidiai). wrap it:

```cmake
function(add_kleidiai)
    enable_language(ASM)
    add_subdirectory(kleidiai)
endfunction()
```

this does not work. `enable_language` inside a function scope sets `CMAKE_ASM_COMPILE_OBJECT` in the function's scope, which evaporates when the function returns. the subdirectory sees ASM as enabled but has no compile rule. cryptic error, no obvious fix in the cmake docs.

move it to directory scope (inline at the top of the kleidiai subdirectory). windows and linux green. ship it.

self-hosted arm runner: red. turns out the runner uses a different ninja version that reorders the enable_language calls relative to `project()`, and now we're back to the original CMP0194 warning but on a job the fork CI never ran.

reverted.

---

## what to take from this

1. **`enable_language` inside a function is broken.** don't do it. use directory scope or the top-level CMakeLists. the cmake issue tracker has threads about this going back years. it is not fixed.

2. **fork CI is not upstream CI.** self-hosted runners almost never run on forks. a green PR on your fork tells you nothing about whether the upstream matrix will pass. separate note on this.

3. **cosmetic warnings are not worth real breakage.** CMP0194 was a warning. the build was working. the cost-benefit of "silence this warning" vs "risk regressing the build matrix" was not in our favor and i didn't think about it hard enough before opening the PR.

4. **ship the revert first, then fix it properly.** do not try to forward-fix a broken master with another speculative PR while people are being blocked. revert, stabilize, then take another crack when you have time to test on every platform.

the lesson i actually wanted to write down: cmake's scoping rules for language enablement are not what you'd guess, and the cost of guessing wrong is a red build matrix in public.
