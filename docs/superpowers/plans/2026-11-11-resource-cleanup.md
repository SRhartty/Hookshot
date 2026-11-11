# Resource cleanup implementation plan

> **For agentic workers:** Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Release temporary Vulkan objects and thread wake events when their owners discard them.

**Architecture:** Keep the existing cache and thread interfaces. Add cleanup only to failed seed variants, precompile pipelines that lose cache insertion, and destroyed joined threads. Test extracted production functions with counted external-resource operations, without a GPU or game.

**Tech stack:** C, C++20, Python standard-library test drivers, Windows and Linux.

**Spec:** The user-approved three-part design in this chat: failed seed variants release their modules; unpublished colliding pipelines are destroyed; thread destruction closes its wake event after the thread has joined.

## Constraints

- No game files, keys, generated game code, binaries, caches, or saves in commits or uploads.
- Preserve the renderer API and shader outputs; no API version bump or generated SPIR-V changes.
- Tests follow the repository's source-extraction convention and run through tests/run_all.py.
- Work from current origin/main, on fix/resource-cleanup-leaks.

## Review focus

- Successful seed variants keep modules alive, while failures and duplicates release temporary modules exactly once.
- Null handles are never passed to counted destruction paths.
- A colliding pipeline is unpublished: destroy the new object, preserve the existing entry and its readiness.
- Thread wake handles are closed only after termination and before owner memory is freed.
- Keep caches and in-flight resource retirement unchanged.

## Task 1: Failed seed variants

- [x] Add tests/test_seed_variant_cleanup.py and an authored C++ fixture extracting seed_variant_of and its actual owner structures.
- [x] Demonstrate failure on unmodified source for partial graphics and mesh translation failures; cover successful insertion and duplicates.
- [x] Modify gpu/vk/vk_shader.cpp to destroy temporary stage modules before its failed-variant return.
- [x] Run the new regression plus program-cache cleanup checks.

## Task 2: Precompile collision

- [x] Add tests/test_precompile_pipeline_cleanup.py and an authored fixture executing the production insertion block with counted pipelines.
- [x] Demonstrate failure when a fresh pipeline loses insertion; cover new insertion and null results.
- [x] Modify gpu/vk/vk_draw.cpp to destroy a non-null unpublished losing pipeline.
- [x] Run the regression plus pipeline-cache checks.

## Task 3: Thread event

- [x] Add tests/test_thread_wake_cleanup.py and an authored fixture executing hos_thread_destroy.
- [x] Demonstrate the unclosed-event failure over repeated lifecycles; cover null handles and registry removal.
- [x] Verify the destroy callers join first, then modify host/hos_thread.c to close its wake handle before free.
- [x] Run the new regression and relevant host lifetime checks.

## Integration and PR

- [x] Review each task and the combined diff independently; resolve substantive findings.
- [x] Run python -I -B tests/run_all.py and record all failures/skips; distinguish environment issues from regressions.
- [x] Verify the diff contains only authored code/tests and this plan; refresh main before publication.
- [ ] Commit with Conventional Commits and AI-assistance trailers; push the contribution branch and create the requested upstream PR.
- [x] State that GPU gameplay, driver memory savings, and Linux runtime execution were not validated locally.

## Validation record

- All three regression drivers pass with LLVM-MinGW clang++ and GCC on Windows.
- Each regression was demonstrated to fail before its corresponding fix.
- Independent combined review found no actionable issues.
- The full local suite completed: 337 passed, 9 failed, 31 skipped. The initial fpx_sticky failure passed after setting HOOKSHOT_MINGW. An untouched 49f75dc8 source export reproduced the remaining dependency, privilege, SDK and recorder-control failures. Its archive-only byte-compile failure is excluded from the comparison because the export has no .git directory; byte-compilation passes in the actual checkout. The stream-visibility check passes on isolated rerun (the baseline run failed deleting its temporary executable). Detailed results are recorded in the PR.
- No game run, GPU/driver memory measurement, Linux runtime execution, renderer API change, or generated shader change is claimed.
