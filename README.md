# codepath-open-source-contribution2

# Open Source Contribution – JupyterLite Issue #155

## Project Information

**Repository:** https://github.com/jupyterlite/jupyterlite

**My Fork:** https://github.com/tiffanyelangwa/jupyterlite

**My Branch:** https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal

**Issue:** https://github.com/jupyterlite/jupyterlite/issues/155

**Issue Title:** Kernel gets disposed before it gets a chance to respond to commInfoRequest

---

## Why I Chose This Issue

I chose JupyterLite issue #155 because it combines technologies that align well with my current programming experience and learning goals. I have experience programming in Python and have been building software projects that involve debugging, understanding existing codebases, and reasoning through application behavior. Contributing to JupyterLite will give me experience working with a mature open-source project while exposing me to how browser-based notebook environments communicate with kernels.

My primary learning goal is to become more comfortable navigating a large production codebase, reproducing reported bugs, understanding the interaction between different software components, and collaborating with maintainers throughout the open-source contribution process.

From reading the issue, I understand that when a notebook is closed, JupyterLite disposes of the kernel before it has finished responding to a `commInfoRequest`. As a result, an asynchronous request is cancelled prematurely, producing an error even though the notebook is simply shutting down. My goal is to reproduce this behavior, identify where the kernel lifecycle is being interrupted, and investigate an appropriate fix.

---

## Problem Summary

The issue occurs when a notebook is closed while the application is still waiting for the kernel to respond to a `commInfoRequest`. Instead of allowing the request to finish gracefully, the kernel is disposed too early, causing the application to throw a **"Canceled future for comm_info_request message before replies were done"** error. Although this happens during shutdown, it creates unnecessary errors and suggests that the kernel shutdown sequence is not being handled correctly. I chose this issue because it is a well-scoped bug that will help me learn about asynchronous programming, kernel lifecycle management, and debugging within a real-world open-source project.

---

## Why This Project

JupyterLite is an actively maintained open-source project with detailed documentation and a clear contribution workflow. The issue is currently open, labeled as both **good first issue** and **help wanted**, making it an appropriate contribution for a new contributor while still requiring meaningful investigation and debugging.

---

## Initial Investigation

From reading the issue discussion, the problem appears to occur during the notebook shutdown process. The kernel receives a `commInfoRequest`, but before it has the opportunity to send its response, the shutdown sequence disposes of the kernel and cancels the pending request. This results in the following error, along with a full stack trace posted in the original issue:

```text
Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done
    at t.KernelShellFutureHandler.dispose (future.js:179)
    at default.js:982
    at Map.forEach (<anonymous>)
    at v._clearKernelState (default.js:981)
    at v.restart (default.js:482)
    at Y.restartKernel (sessioncontext.js:298)
    at Object.restart (sessioncontext.js:758)
```

This stack trace provides a strong starting point for investigating where the bug originates (see **Root Cause Investigation** below).

---

## Environment Setup Challenges

During setup I encountered several issues that required investigation.

### 1. Missing Yarn

Initially, the `yarn` command was unavailable because Corepack had not been enabled.

**Error**

```text
yarn: command not found
```

**Resolution**

I enabled Corepack and activated the repository's required Yarn version by running:

```bash
corepack enable
corepack prepare yarn@3.5.0 --activate
```

After doing so, the `yarn` command became available.

### 2. Missing `jlpm`

After installing the project dependencies, the build process failed with:

```text
jlpm: command not found
```

The repository's build scripts reference `jlpm`, but the command was unavailable in my development environment despite following the setup instructions in `CONTRIBUTING.md`.

To investigate, I:

- Confirmed that the build scripts in `package.json` reference `jlpm`.
- Verified that the repository specifies Yarn 3.5.0 through its `packageManager` field.
- Checked the official setup documentation to ensure I had followed the recommended installation steps.
- Investigated whether `jlpm` should be provided by the project's Node.js tooling or by a Python package installation.

At the time of writing, I have not yet fully resolved why `jlpm` is unavailable in my environment. This will be my next step before reproducing the issue locally and beginning implementation.

---

## Expected Behavior

When a notebook is closed, the kernel should shut down gracefully. Any pending kernel communication, including a `commInfoRequest`, should either complete successfully or be cleaned up without producing uncaught errors in the browser console.

---

## Actual Behavior

According to the original issue report, the kernel is disposed before the pending `commInfoRequest` receives a response. The following uncaught error appears in the browser console:

```text
Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done
```

According to the issue report, this is logged as an uncaught promise rejection rather than a handled shutdown message, indicating unintended behavior during kernel shutdown.

---

## Root Cause Investigation

Reading the stack trace from the original issue line by line suggests that this is not a bug in JupyterLite's own codebase. Instead, it appears to originate in the `@jupyterlab/services` and `@jupyterlab/apputils` packages that JupyterLite depends on and reuses for kernel and session management.

- **sessioncontext.js** (compiled from `packages/apputils/src/sessioncontext.tsx` in `jupyterlab/jupyterlab`) - `SessionContext.restart()` is called, which triggers the session manager to restart or shut down the kernel connection.

- **default.js** (compiled from `packages/services/src/kernel/default.ts`) - Inside `KernelConnection.restart()`, the private method `_clearKernelState()` runs. This method loops over `this._futures` (the map of all pending, in-flight kernel requests) and calls `future.dispose()` on every single one, regardless of whether that future has already received its final reply.

- **future.js** (compiled from `packages/services/src/kernel/future.ts`) - `KernelFutureHandler.dispose()` is what ultimately throws **"Canceled future for ... message before replies were done"** when a future is disposed while it is still awaiting a reply.

Based on the available issue discussion and stack trace, the likely root cause appears to be that `_clearKernelState()` treats all pending futures as safe to tear down during shutdown or restart without first checking whether a given future is still expecting a reply. The `commInfoRequest` future, which is naturally still in flight because it is issued to set up comm targets for widgets, gets caught in that blanket disposal sweep and throws.

This hypothesis is stronger than simply stating that "the shutdown order is wrong" because it identifies the specific function performing the unconditional operation (`this._futures.forEach(future => future.dispose())`) that appears responsible for the observed behavior. I plan to confirm this hypothesis by reproducing the issue locally once the development environment is fully operational.

---

## Supporting Evidence (Analogous Bug)

I found a very similar report in a related project: **jupyter-resource-usage#172**. It shows the same stack trace pattern (`_clearKernelState()` → cancelled future), although this time for `kernel_info_request` rather than `comm_info_request`, and triggered by clicking **Restart Kernel and Run All Cells** instead of closing a notebook.

This provides additional evidence that the likely root cause is a general property of `_clearKernelState()`'s unconditional future-disposal loop rather than something specific to notebook closure. Although I have not yet verified this experimentally, the similarity between the two stack traces suggests they originate from the same underlying behavior.

---

## Files to Investigate / Modify

Based on the stack trace and the analogous bug above, the following files and functions appear to be the most relevant areas to investigate:

- **packages/services/src/kernel/default.ts** - `KernelConnection._clearKernelState()`. This is where the unconditional `future.dispose()` loop lives. A fix would need to either:
  - await or flush any futures that are expected to resolve imminently before clearing state, or
  - mark futures disposed during a normal shutdown or restart differently from futures cancelled mid-flight for other reasons, so the error is not thrown in the former case.

- **packages/services/src/kernel/future.ts** - `KernelFutureHandler.dispose()`. This is where the error message is actually thrown. Any fix to suppress or handle this gracefully during expected shutdown would likely require a corresponding change here, such as passing a flag from `_clearKernelState()` indicating that the disposal is part of a normal shutdown rather than a genuine cancellation.

- **packages/apputils/src/sessioncontext.tsx** - `SessionContext.restart()` and the dispose path invoked on notebook close. It is worth investigating whether shutdown could instead be sequenced to wait briefly for in-flight comm setup before tearing down the kernel connection as an alternative to modifying `_clearKernelState()` itself.

**Important scope note:** Since `default.ts`, `future.ts`, and `sessioncontext.tsx` all live in the upstream `jupyterlab/jupyterlab` repository through the `@jupyterlab/services` and `@jupyterlab/apputils` packages that JupyterLite consumes as dependencies, the eventual code fix may need to be proposed as a pull request against **jupyterlab/jupyterlab** rather than **jupyterlite/jupyterlite**. Alternatively, JupyterLite could potentially work around the issue by avoiding a `commInfoRequest` during teardown or by explicitly awaiting it before triggering kernel disposal. I plan to raise this scope question with the maintainers during Phase III before deciding where the implementation belongs.

## Root Cause Investigation

Reading the stack trace from the original issue line by line suggests that this is not a bug in JupyterLite's own codebase. Instead, the error appears to originate in the `@jupyterlab/services` and `@jupyterlab/apputils` packages that JupyterLite depends on and reuses for kernel and session management.

- **sessioncontext.js** (compiled from `packages/apputils/src/sessioncontext.tsx` in `jupyterlab/jupyterlab`) - `SessionContext.restart()` is called, which triggers the session manager to restart or shut down the kernel connection.

- **default.js** (compiled from `packages/services/src/kernel/default.ts`) - inside `KernelConnection.restart()`, the private method `_clearKernelState()` runs. This method loops over `this._futures` (the map of all pending, in-flight kernel requests) and calls `future.dispose()` on each one, regardless of whether the future has already received its final reply.

- **future.js** (compiled from `packages/services/src/kernel/future.ts`) - `KernelFutureHandler.dispose()` is what ultimately throws the error:

```
Canceled future for comm_info_request message before replies were done
```

when a future is disposed while it is still awaiting a reply.

Based on the available issue discussion and stack trace, the likely root cause is that `_clearKernelState()` treats all pending futures as safe to dispose during shutdown or restart without first checking whether they are still awaiting responses. The `commInfoRequest` future, which is naturally in flight while widget communication is being established, appears to be caught in this blanket disposal process and is cancelled prematurely.

This is currently a hypothesis based on the available evidence from the issue discussion and stack trace. I plan to verify it by reproducing the issue locally once my development environment is fully operational.

## Supporting Evidence (Analogous Bug)

While investigating, I found a similar report in another project: **jupyter-resource-usage#172**. That issue produces nearly the same stack trace, except the cancelled request is `kernel_info_request` instead of `comm_info_request`, and it is triggered by restarting the kernel rather than closing a notebook.

Although the trigger differs, both reports involve `_clearKernelState()` disposing pending kernel futures during shutdown or restart. This suggests the underlying behavior may not be specific to notebook closure or widget communication, but instead may stem from the shared kernel cleanup logic.

I have not yet verified this experimentally, but the similarity between the two stack traces provides additional evidence that they may share the same root cause.

## Files to Investigate / Modify

Based on the stack trace and the analogous issue above, the following files appear to be the most relevant:

- **packages/services/src/kernel/default.ts**

  `KernelConnection._clearKernelState()`

  This method contains the unconditional `future.dispose()` loop. A potential fix could involve waiting for futures that are expected to complete naturally before clearing kernel state, or distinguishing between normal shutdown and unexpected cancellation.

- **packages/services/src/kernel/future.ts**

  `KernelFutureHandler.dispose()`

  This is where the cancellation error is thrown. Another possible approach would be to suppress or handle this specific disposal differently during an expected shutdown sequence.

- **packages/apputils/src/sessioncontext.tsx**

  `SessionContext.restart()`

  This file coordinates the kernel restart and shutdown process. It may be possible to adjust the shutdown sequence so that pending communication is allowed to complete before disposing of the kernel connection.

### Scope Note

One important observation from my investigation is that the functions identified above live in the upstream **JupyterLab** repository rather than the JupyterLite repository itself. Since JupyterLite reuses these packages, the actual code fix may ultimately belong in **jupyterlab/jupyterlab** instead of **jupyterlite/jupyterlite**.

Alternatively, JupyterLite may be able to work around the issue by changing how or when it issues `commInfoRequest` during notebook shutdown.

I plan to clarify this with the project maintainers during Phase III before implementing a fix.

## Solution Plan (UMPIRE)

### U — Understand

The reported bug occurs when a notebook is closed while a `commInfoRequest` is still awaiting a response from the kernel.

Based on the available stack trace, `KernelConnection._clearKernelState()` appears to dispose all pending futures during shutdown or restart without checking whether they are still awaiting replies. This appears to cause the in-flight `commInfoRequest` future to throw a cancellation error through `KernelFutureHandler.dispose()`.

### M — Match

This resembles a common asynchronous cleanup problem where a shared resource is disposed while another component is still waiting to use it.

The analogous `jupyter-resource-usage#172` issue further suggests that this behavior affects multiple request types and likely originates from the shared kernel cleanup logic rather than widget-specific code.

### P — Plan

- Reproduce the issue locally using a widget-enabled notebook.
- Verify that the browser console produces the same stack trace.
- Confirm whether `_clearKernelState()` is disposing pending futures before replies complete.
- Determine whether the fix belongs in `_clearKernelState()`, `KernelFutureHandler.dispose()`, or elsewhere in the shutdown sequence.
- Confirm with maintainers whether the fix should be submitted to JupyterLite or JupyterLab.
- Implement the fix on the `fix-issue-155-kernel-disposal` branch.

### I — Implement

Implementation will take place during Phase III after reproducing the issue locally and confirming the correct repository for the fix.

### R — Review

After implementing the fix, I will verify that:

- notebook shutdown completes normally,
- no `commInfoRequest` cancellation error appears in the browser console,
- kernel shutdown continues to function correctly, and
- existing kernel restart functionality is not negatively affected.

### E — Evaluate

The fix will be considered successful if notebook shutdown completes without producing the cancellation error while preserving normal kernel restart and notebook functionality.

## Additional Investigation

The issue was originally reported in 2021 and remains open, indicating that it has persisted across multiple releases of JupyterLite.

My investigation also suggests that the relevant code lives within upstream JupyterLab packages, meaning the eventual fix may need to be contributed there rather than directly to JupyterLite.

While setting up the development environment, I encountered tooling issues related to the project's build process, particularly the missing `jlpm` command despite following the documented setup instructions. Resolving this environment issue is my immediate next step so that I can reproduce the bug locally, validate the hypotheses developed during my investigation, and begin implementing a fix.
