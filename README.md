
# codepath-open-source-contribution2
# Open Source Contribution – JupyterLite Issue #155
 
## Project Information
 
**Repository:** https://github.com/jupyterlite/jupyterlite
 
**My Fork:** https://github.com/tiffanyelangwa/jupyterlite
 
**My Branch:** https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal
 
**Issue:** https://github.com/jupyterlite/jupyterlite/issues/155
 
**Issue Title:** Kernel gets disposed before it gets a chance to respond to commInfoRequest
 
---
 
# Why I Chose This Issue
 
I chose JupyterLite issue #155 because it combines technologies that align well with my current programming experience and learning goals. I have experience programming in Python and have been building software projects that involve debugging, understanding existing codebases, and reasoning through application behavior. Contributing to JupyterLite will give me experience working with a mature open-source project while exposing me to how browser-based notebook environments communicate with kernels.
 
My primary learning goal is to become more comfortable navigating a large production codebase, reproducing reported bugs, understanding the interaction between different software components, and collaborating with maintainers throughout the open-source contribution process.
 
From reading the issue, I understand that when a notebook is closed, JupyterLite disposes of the kernel before it has finished responding to a `commInfoRequest`. As a result, an asynchronous request is cancelled prematurely, producing an error even though the notebook is simply shutting down. My goal is to reproduce this behavior, identify where the kernel lifecycle is being interrupted, and investigate an appropriate fix.
 
---
 
# Problem Summary
 
The issue occurs when a notebook is closed while the application is still waiting for the kernel to respond to a `commInfoRequest`. Instead of allowing the request to finish gracefully, the kernel is disposed too early, causing the application to throw a "Canceled future for comm_info_request message before replies were done" error. Although this happens during shutdown, it creates unnecessary errors and suggests that the kernel shutdown sequence is not being handled correctly. I chose this issue because it is a well-scoped bug that will help me learn about asynchronous programming, kernel lifecycle management, and debugging within a real-world open-source project.
 
---
 
# Why This Project
 
JupyterLite is an actively maintained open-source project with detailed documentation and a clear contribution workflow. The issue is currently open, labeled as both **good first issue** and **help wanted**, making it an appropriate contribution for a new contributor while still requiring meaningful investigation and debugging.
 
---
 
# Initial Investigation
 
From reading the issue discussion, the problem appears to occur during the notebook shutdown process. The kernel receives a `commInfoRequest`, but before it has the opportunity to send its response, the shutdown sequence disposes of the kernel and cancels the pending request. This results in the following error, along with a full stack trace posted in the original issue:
 
```
Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done
    at t.KernelShellFutureHandler.dispose (future.js:179)
    at default.js:982
    at Map.forEach (<anonymous>)
    at v._clearKernelState (default.js:981)
    at v.restart (default.js:482)
    at Y.restartKernel (sessioncontext.js:298)
    at Object.restart (sessioncontext.js:758)
```
 
This stack trace is the key to identifying exactly where the bug lives (see "Root Cause Investigation" below).
 
---
 
# Environment Setup
 
## Development Branch
 
I created and pushed a dedicated development branch for this issue:
 
**Branch:** `fix-issue-155-kernel-disposal`
**Link:** https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal
 
This branch will be used for all investigation and implementation related to Issue #155.
 
## Setup Approach
 
I followed the official JupyterLite development setup instructions provided in the project's `CONTRIBUTING.md` file (README/documented setup path) rather than using a development container. I installed the required versions of Node.js and Python, cloned my fork of the repository, installed the project dependencies, and attempted to build the project according to the documented setup process.
 
## Environment Setup Challenges
 
During setup I encountered several issues that required investigation:
 
1. Initially, the `yarn` command was unavailable because Corepack had not been enabled. **Error:** `yarn: command not found`. **Fix:** I resolved this by running `corepack enable` and then `corepack prepare yarn@3.5.0 --activate`, which matches the Yarn version pinned in the repository's `package.json`.
 
2. After successfully installing project dependencies, the build process failed with `jlpm: command not found`. The repository's build scripts call `jlpm`, but no `jlpm` executable had been created on my PATH despite following the official setup instructions. I investigated by:
   - Confirming the build scripts (`package.json` scripts section) reference `jlpm` directly.
   - Running `pip show jupyterlab` to verify JupyterLab was actually installed in my active environment, since `jlpm` is provided as a wrapper script bundled with the JupyterLab Python package, not by `yarn`/`npm`.
   - Finding that my JupyterLab install had gone into a different virtual environment than the one I was running commands from, so its `bin/jlpm` script was never on my active PATH.
   - **Fix:** activating the correct virtual environment (the one where `jupyterlab` was actually installed) resolved the missing `jlpm` command.
 
---
 
# Bug Reproduction
 
## Steps to Reproduce
 
1. Launch a JupyterLite notebook in the browser (e.g., via the hosted demo or a local `jupyter lite serve` build).
2. Open a notebook that includes at least one `ipywidgets` widget (widgets are what trigger comm channel setup, which is what makes `commInfoRequest` relevant here).
3. Wait for the kernel to start and confirm the widget renders successfully.
4. Close the notebook tab/panel while the kernel is still active (i.e., don't wait for it to idle out or manually shut the kernel down first).
5. Open the browser's developer console and observe the output during/after notebook shutdown.
 
## Expected Behavior
 
The notebook should close gracefully, allowing any pending kernel communication (including the `commInfoRequest`) to complete before the kernel is disposed. No errors should appear in the console during shutdown.
 
## Actual Behavior
 
The kernel is disposed before the pending `commInfoRequest` receives a response. The following uncaught error appears in the console:
 
```
Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done
```
 
This is logged as an uncaught promise rejection, not a handled/expected shutdown message — confirming this is a bug rather than intended behavior.
 
---
 
# Root Cause Investigation
 
Reading the stack trace from the original issue line by line shows this is **not a bug in JupyterLite's own codebase** — it originates in the `@jupyterlab/services` and `@jupyterlab/apputils` packages that JupyterLite depends on and reuses for kernel/session management:
 
1. `sessioncontext.js` (compiled from `packages/apputils/src/sessioncontext.tsx` in `jupyterlab/jupyterlab`) — `SessionContext.restart()` is called, which triggers `sessionManager` to restart/shut down the kernel connection.
2. `default.js` (compiled from `packages/services/src/kernel/default.ts`) — inside `KernelConnection.restart()`, the private method **`_clearKernelState()`** runs. This method loops over `this._futures` (the map of all pending, in-flight kernel requests) and calls `future.dispose()` on **every single one, unconditionally** — regardless of whether that future has already received its final reply.
3. `future.js` (compiled from `packages/services/src/kernel/future.ts`) — `KernelFutureHandler.dispose()` is what actually throws `"Canceled future for ... message before replies were done"` when a future is disposed while it is still awaiting a reply.
 
**Root cause:** `_clearKernelState()` treats all pending futures as safe to tear down during shutdown/restart, without first checking whether a given future is still expecting a reply. The `commInfoRequest` future — which is naturally in-flight at close time because it's issued to set up comm targets for widgets — gets caught in that same blanket disposal sweep and throws.
 
This is a stronger root-cause claim than "the shutdown order is wrong," because it points to the exact function performing an unconditional operation (`this._futures.forEach(future => future.dispose())`) with no completion check, rather than describing symptoms.
 
## Supporting evidence (analogous bug)
 
I found a very similar report in a related project: [jupyter-resource-usage#172](https://github.com/jupyter-server/jupyter-resource-usage/issues/172) shows the **exact same stack trace shape** (`_clearKernelState` → cancelled future, this time for `kernel_info_request` instead of `comm_info_request`), but triggered by clicking "Restart kernel and run all cells" rather than closing a notebook. This confirms the root cause is a general property of `_clearKernelState()`'s unconditional future-disposal loop, not something specific to comms or to the notebook-close code path — the same underlying method misbehaves whenever *any* kernel restart/shutdown races against *any* in-flight request.
 
---
 
# Files to Investigate / Modify
 
Based on the stack trace and the analogous bug above, the following files and functions are directly implicated:
 
- **`packages/services/src/kernel/default.ts`** — `KernelConnection._clearKernelState()`. This is where the unconditional `future.dispose()` loop lives. A fix would need to either (a) await/flush any futures that are expected to resolve imminently before clearing state, or (b) mark futures disposed during a normal shutdown/restart differently from futures cancelled mid-flight for other reasons, so the error isn't thrown in the former case.
- **`packages/services/src/kernel/future.ts`** — `KernelFutureHandler.dispose()`. This is where the error message is actually thrown. Any fix to suppress/handle this gracefully during expected shutdown would need a corresponding change here (e.g., a flag passed in from `_clearKernelState()` indicating "this is a normal-shutdown disposal, not a genuine cancellation").
- **`packages/apputils/src/sessioncontext.tsx`** — `SessionContext.restart()` / the dispose path invoked on notebook close. Worth checking whether shutdown could be sequenced to wait briefly for in-flight comm setup before tearing down the kernel connection, as an alternative fix location to changing `_clearKernelState()` itself.
 
**Important scope note:** since `default.ts`, `future.ts`, and `sessioncontext.tsx` all live in the upstream `jupyterlab/jupyterlab` repository (via the `@jupyterlab/services` and `@jupyterlab/apputils` packages that JupyterLite consumes as a dependency), the actual code fix may need to be proposed as a PR against `jupyterlab/jupyterlab` rather than `jupyterlite/jupyterlite`. Alternatively, JupyterLite could work around the issue on its own side by not issuing a `commInfoRequest` during teardown, or by explicitly awaiting it before triggering kernel disposal. I plan to raise this scope question with maintainers in Phase III before deciding which repository the fix belongs in.
 
---
 
# Solution Plan (UMPIRE)
 
## U — Understand
 
The reported bug occurs when a notebook is closed while a `commInfoRequest` is still awaiting a response from the kernel. `KernelConnection._clearKernelState()` in `default.ts` disposes all pending futures unconditionally during shutdown/restart, without checking whether they're still awaiting a reply, so the in-flight `commInfoRequest` future throws a "Canceled future" error via `KernelFutureHandler.dispose()` in `future.ts`.
 
## M — Match
 
This resembles a classic "resource torn down while a consumer still holds a live reference" bug, common in async cleanup code. The analogous [jupyter-resource-usage#172](https://github.com/jupyter-server/jupyter-resource-usage/issues/172) issue confirms the same `_clearKernelState()` method throws for `kernel_info_request` under a different trigger (kernel restart from the UI), meaning the fix belongs in the shared disposal path, not in comm-specific or close-specific code.
 
## P — Plan
 
1. Reproduce the issue locally in a built JupyterLite site with a widget-containing notebook.
2. Confirm via browser dev tools that the trace matches `_clearKernelState()` → `future.dispose()`.
3. Decide whether the fix should live in `_clearKernelState()` (skip/await in-flight futures before disposal) or in `future.ts` (suppress the thrown error for shutdown-triggered disposal specifically).
4. Determine whether this needs to be filed/fixed upstream in `jupyterlab/jupyterlab`, or worked around locally in JupyterLite's own session/kernel handling code.
5. Implement the chosen fix on the `fix-issue-155-kernel-disposal` branch.
 
## I — Implement
 
Implementation will occur during Phase III, once the repository scope question (JupyterLab vs. JupyterLite) is confirmed with maintainers.
 
## R — Review
 
After implementing the fix, I will verify that:
 
- the notebook closes normally with a widget-containing notebook open,
- no `commInfoRequest` cancellation error appears in the console,
- kernel shutdown still completes successfully, and
- existing kernel restart functionality (the code path shared with the jupyter-resource-usage#172 case) still works correctly, i.e. the fix doesn't just suppress the error superficially.
 
## E — Evaluate
 
The fix will be considered successful if the browser console no longer reports the cancellation error during notebook close, notebook and kernel-restart shutdown behavior both remain stable, and no regressions are introduced in normal notebook use.
 
---
 
# Additional Investigation
 
The issue was originally reported in 2021 and remains open, indicating it has persisted across multiple releases of JupyterLite (and, since the root cause lives upstream, across multiple releases of JupyterLab as well). While investigating the development environment, I also encountered tooling changes related to the project's build system (`jlpm`), suggesting that parts of the development workflow have evolved over time. This reinforces the importance of understanding both the current architecture and the historical evolution of the project's kernel lifecycle before implementing a fix.
 
**Next step for deeper investigation:** run `git log --oneline --follow -- packages/services/src/kernel/default.ts` against a clone of `jupyterlab/jupyterlab` to find when the unconditional `future.dispose()` loop in `_clearKernelState()` was introduced, to check whether it was a deliberate simplification (i.e., a scope gap rather than an outright bug) or a regression.
 
---
 
# Phase III: Implementation
 
## Updated Root Cause (revised from Phase II)
 
Phase II's investigation, based only on the stack trace, concluded the root cause lived upstream in `@jupyterlab/services`' `_clearKernelState()`, which disposes all pending futures unconditionally on kernel shutdown/restart. That conclusion wasn't wrong, but deeper investigation in Phase III — reading the actual dispatcher code in this repository — found a more specific and directly fixable cause **inside JupyterLite's own codebase**:
 
`packages/services/src/kernel/base.ts`'s `BaseKernel.handleMessage()` dispatcher is a `switch` statement that handles every incoming shell request type (`kernel_info_request`, `execute_request`, `complete_request`, `history_request`, `comm_open`/`comm_msg`/`comm_close`, etc.) — **except `comm_info_request`**, which falls through to `default: break` and is silently dropped. No `comm_info_reply` is ever sent. The abstract method `commInfoRequest(content)` was already declared on `BaseKernel` (and every concrete kernel subclass is already required by TypeScript to implement it), but nothing in the dispatcher ever called it.
 
This means the `commInfoRequest` future sent by `@jupyterlab/services` was never going to resolve, regardless of shutdown timing. It only *looked* like a timing/race bug because the "Canceled future" error only surfaces once the kernel is eventually disposed on notebook close/restart, which is what `_clearKernelState()` does. The disposal is still the proximate trigger of the error message, but the actual defect — the thing that made the request unresolvable in the first place — is the missing dispatcher case in `base.ts`.
 
## Implementation Progress
 
- **`packages/services/src/kernel/base.ts`** (commit [`c5b35671`](https://github.com/tiffanyelangwa/jupyterlite/commit/c5b35671a626f219f2b2afd0671bd11136e4fdcb)) — added the missing `case 'comm_info_request':` to the `handleMessage` switch statement, and a new private `_commInfo()` method (mirroring the existing `_complete()` / `_inspect()` handlers) that calls the already-required abstract `commInfoRequest(content)` and sends a `comm_info_reply` on the shell channel with `parentHeader` set to the request's header, so `@jupyterlab/services`' `KernelShellFutureHandler` can match the reply to its pending future and resolve it normally.
- **`ui-tests/test/kernels.spec.ts`** (commit [`d9d6dba3`](https://github.com/tiffanyelangwa/jupyterlite/commit/d9d6dba321e5fc5f9a9ee18d15fc28849df837b9)) — added a new, standalone Playwright test, `"Closing a notebook does not cancel comm_info_request"`, placed directly after the existing "Kernel shutdown" test. It follows that test's existing conventions (`page.notebook.createNew()`, `page.notebook.save()`, `page.notebook.close(true)`), attaches a `page.on('console', ...)` listener filtering for `comm_info_request` errors during that flow, and asserts none are logged.
 
Branch: [`fix-issue-155-kernel-disposal`](https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal) — both commits are visible there.
 
## Testing
 
**Automated (compile-level):** ran `tsc -b` on `packages/services` — exit code 0, no errors. Confirmed the compiled output (`lib/kernel/base.js`, `lib/kernel/base.d.ts`) contains the new `comm_info_request` case, `_commInfo` method, and `comm_info_reply` message construction, so the fix is present in what actually ships.
 
**Automated (end-to-end, attempted but not completed):** attempted to run the new Playwright test (`ui-tests/test/kernels.spec.ts`) against a full local JupyterLite build to get a genuine pass/fail result rather than relying on manual tracing. This required rebuilding the whole app from source (`jlpm build` → `jlpm pack:app` → `ui-tests`' own `jlpm build` → `playwright install chromium` → running the suite), which surfaced a chain of local environment issues (documented in detail under "Challenges Faced" below) rooted in this machine's conda-forge JupyterLab install missing its entire `jlpm` toolchain (`jlpmapp.py`, the bundled `staging/yarn.js`, and the `jlpm` entry point). I made the deliberate call to stop investigating this rather than start modifying global Yarn plugin configuration to route around a local install gap, since it isn't related to the correctness of the fix itself. This is a real limitation of my current testing evidence: the new test has not actually been executed and observed to pass, only manually verified against the compiled dispatcher logic.
 
**Manual verification:** traced the compiled fix step by step against the message flow described in Phase II's root-cause investigation — confirmed `handleMessage` now invokes `commInfoRequest` and constructs a `comm_info_reply` with the correct `parentHeader`/`session`, which is exactly what `KernelShellFutureHandler` needs to resolve the previously-pending future instead of leaving it to be cancelled on disposal.
 
**Existing test suite:** did not run the full existing Playwright suite for the same environment reasons above. The `tsc -b` compile check across the `services` package passed cleanly, which would catch any type-level regression the change introduced.
 
## Challenges Faced
 
**1. Locating the real bug (not just the reported symptom).** The original issue report and Phase II's stack-trace-only investigation both frame this as a shutdown-timing race. Reading the actual `handleMessage` dispatcher in `base.ts` showed this framing is incorrect: `comm_info_request` was never wired into the dispatcher at all, so the request could never succeed regardless of timing. This was resolved by grepping the full repo for `commInfoRequest` and `_commInfo`-style handlers and comparing the dispatcher's `switch` cases against the full set of request types `@jupyterlab/services` can send.
 
**2. A chain of local build-tooling gaps when attempting to run the new test end-to-end.**
   - Corepack initially didn't have `yarn` enabled (same class of issue as Phase II's setup).
   - The project's build/test scripts call `jlpm`, which turned out to be completely absent from this machine (not just off `PATH`) — a `jlpm` → `yarn` shim was created to work around simple script resolution.
   - The shim was insufficient for the actual build, because `jlpm build` calls `yarn workspaces foreach`, which requires the `plugin-workspace-tools` Yarn plugin. That plugin is normally bundled inside the real `jlpm`'s own vendored Yarn distribution — plain Corepack `yarn` doesn't include it.
   - Root cause: this machine's conda-forge JupyterLab install is a trimmed build missing `jlpmapp.py`, the bundled `staging/yarn.js`, and the `jlpm` console-script entry point entirely, which a standard PyPI JupyterLab wheel normally ships.
   - **Resolution:** rather than modify global Yarn plugin configuration or reinstall JupyterLab from PyPI (both of which affect things outside the scope of this fix), I stopped once the root cause was clearly identified and documented the path forward (installing a standard JupyterLab, or importing `workspace-tools` at the home-level Yarn config) for whoever picks this up next, and relied on the `tsc` compile check plus manual trace as the verification for this submission.
 
## Process Note: AI Assistance Disclosure
 
This phase's implementation was completed with assistance from Claude Code (Anthropic), used for repository investigation, root-cause tracing, drafting the fix and regression test, and diagnosing the local build-tooling environment. All AI-suggested code changes were reviewed and approved individually before being applied, and each commit includes a `Co-Authored-By` trailer reflecting this.
 
 
---
---
 
# Phase III — Implementation
 
## Implementation Progress
 
- **`packages/services/src/kernel/base.ts`** (commit `c5b35671`)
  - **Line ~129** — added the missing `case 'comm_info_request': await this._commInfo(msg); break;` to the `handleMessage` dispatcher's switch statement, between the existing `complete_request` and `history_request` cases.
  - **Lines ~622–639** — added a new private `_commInfo(msg)` method that mirrors the existing `_complete` / `_inspect` handlers: it casts the incoming message to `ICommInfoRequestMsg`, calls the already-required abstract `commInfoRequest(content)` method (which every concrete kernel implementation must already provide), and builds/sends a `comm_info_reply` message on the shell channel with the correct `parentHeader`, `channel`, and `session`.
  - Previously, the dispatcher fell through to `default: break` for this message type, so no reply was ever sent — meaning the future was never going to resolve regardless of kernel shutdown timing.
 
- **`ui-tests/test/kernels.spec.ts`** (commit `d9d6dba3`)
  - **Lines ~85–106** — added a new Playwright test, `"Closing a notebook does not cancel comm_info_request"`, placed directly after the existing `"Kernel shutdown"` test (lines 49–83). It listens for console errors matching `comm_info_request` during a create → save → close flow and asserts none are logged.
 
---
 
## Testing
 
**Automated (CI):** Pushed commit `829f4fb7` to the branch, which triggered GitHub Actions on the fork. The **UI Tests** workflow (run [`29882450077`](https://github.com/tiffanyelangwa/jupyterlite/actions/runs/29882450077)) ran the full Playwright UI test suite, including the new regression test, and **passed with a green check** (23m 41s runtime). This confirms the fix works end-to-end in a real built JupyterLite environment, not just at the type-check level.
 
**Automated (local):** Ran `tsc -b` on `packages/services` — exit code 0. Verified the compiled output (`lib/kernel/base.js`) contains the new `comm_info_request` handling (5 matches for the new code).
 
**Manual:** Traced the fix through the dispatcher logic and compiled output to confirm `commInfoRequest` is now invoked and a reply is sent on the shell channel with the correct `parentHeader`, which is what allows JupyterLab's `KernelShellFutureHandler` to resolve the future instead of leaving it pending until disposal cancels it.
 
---
 
## Challenges Faced
 
**Root cause revision.** The original hypothesis (from Phase II) was that the root cause lived upstream in `@jupyterlab/services`'s `_clearKernelState()`, which disposes all pending futures unconditionally on kernel shutdown. Deeper investigation in Phase III showed this was only half the story: the real root cause was inside JupyterLite's own `packages/services/src/kernel/base.ts`, where the `handleMessage` dispatcher never had a case for `comm_info_request` and so never sent a reply at all — meaning the future was never going to resolve regardless of shutdown timing. This was resolved by locating the dispatcher's switch statement and confirming, via a full-repo search, that no `_commInfo`-style handler existed for this message type, unlike every other request type.
 
**Working-tree hygiene.** A stray `package-lock.json` and an unrelated `yarn.lock` modification appeared in the working tree from earlier `npm`/`yarn` interaction. These were reverted/removed before committing so the diff stayed scoped to the actual fix.
 
**Local Playwright run blocked by environment, worked around via CI.** Attempting to run the full Playwright suite locally hit a chain of environment issues: Corepack provided `yarn` but not `jlpm`; `jlpm` itself was entirely absent (not just off PATH); a shim pointing `jlpm` at `yarn` resolved the command but `jlpm build` calls `yarn workspaces foreach`, which requires the `plugin-workspace-tools` Yarn plugin, which wasn't installed. Root cause: this machine's conda-forge JupyterLab 4.6.1 install is missing `jlpmapp.py`, `staging/yarn.js`, and the `jlpm` entry point entirely — a standard PyPI wheel ships all three; the conda build is trimmed. Rather than keep modifying global Yarn/plugin config chasing a single local test run, I stopped and instead pushed the branch to trigger GitHub Actions, which ran the real Playwright suite in a correctly-provisioned environment and returned a green result — a stronger verification than a local run would have given anyway.
