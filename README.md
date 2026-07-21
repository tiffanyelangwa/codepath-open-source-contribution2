codepath-open-source-contribution2
Open Source Contribution – JupyterLite Issue #155
Project Information

Repository: https://github.com/jupyterlite/jupyterlite

My Fork: https://github.com/tiffanyelangwa/jupyterlite

My Branch: https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal

Issue: https://github.com/jupyterlite/jupyterlite/issues/155

Issue Title: Kernel gets disposed before it gets a chance to respond to commInfoRequest

Why I Chose This Issue

I chose JupyterLite issue #155 because it combines technologies that align well with my current programming experience and learning goals. I have experience programming in Python and have been building software projects that involve debugging, understanding existing codebases, and reasoning through application behavior. Contributing to JupyterLite will give me experience working with a mature open-source project while exposing me to how browser-based notebook environments communicate with kernels.

My primary learning goal is to become more comfortable navigating a large production codebase, reproducing reported bugs, understanding the interaction between different software components, and collaborating with maintainers throughout the open-source contribution process.

From reading the issue, I understand that when a notebook is closed, JupyterLite disposes of the kernel before it has finished responding to a commInfoRequest. As a result, an asynchronous request is cancelled prematurely, producing an error even though the notebook is simply shutting down. My goal is to reproduce this behavior, identify where the kernel lifecycle is being interrupted, and investigate an appropriate fix.

Problem Summary

The issue occurs when a notebook is closed while the application is still waiting for the kernel to respond to a commInfoRequest. Instead of allowing the request to finish gracefully, the kernel is disposed too early, causing the application to throw a "Canceled future for comm_info_request message before replies were done" error. Although this happens during shutdown, it creates unnecessary errors and suggests that the kernel shutdown sequence is not being handled correctly. I chose this issue because it is a well-scoped bug that will help me learn about asynchronous programming, kernel lifecycle management, and debugging within a real-world open-source project.

Why This Project

JupyterLite is an actively maintained open-source project with detailed documentation and a clear contribution workflow. The issue is currently open, labeled as both good first issue and help wanted, making it an appropriate contribution for a new contributor while still requiring meaningful investigation and debugging.

Initial Investigation

From reading the issue discussion, the problem appears to occur during the notebook shutdown process. The kernel receives a commInfoRequest, but before it has the opportunity to send its response, the shutdown sequence disposes of the kernel and cancels the pending request. This results in the following error, along with a full stack trace posted in the original issue:

Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done
    at t.KernelShellFutureHandler.dispose (future.js:179)
    at default.js:982
    at Map.forEach (<anonymous>)
    at v._clearKernelState (default.js:981)
    at v.restart (default.js:482)
    at Y.restartKernel (sessioncontext.js:298)
    at Object.restart (sessioncontext.js:758)

This stack trace is the key to identifying exactly where the bug lives (see "Root Cause Investigation" below).

Environment Setup
Development Branch

I created and pushed a dedicated development branch for this issue:

Branch: fix-issue-155-kernel-disposal Link: https://github.com/tiffanyelangwa/jupyterlite/tree/fix-issue-155-kernel-disposal

This branch will be used for all investigation and implementation related to Issue #155.

Setup Approach

I followed the official JupyterLite development setup instructions provided in the project's CONTRIBUTING.md file (README/documented setup path) rather than using a development container. I installed the required versions of Node.js and Python, cloned my fork of the repository, installed the project dependencies, and attempted to build the project according to the documented setup process.

Environment Setup Challenges

During setup I encountered several issues that required investigation:

Initially, the yarn command was unavailable because Corepack had not been enabled. Error: yarn: command not found. Fix: I resolved this by running corepack enable and then corepack prepare yarn@3.5.0 --activate, which matches the Yarn version pinned in the repository's package.json.
After successfully installing project dependencies, the build process failed with jlpm: command not found. The repository's build scripts call jlpm, but no jlpm executable had been created on my PATH despite following the official setup instructions. I investigated by:
Confirming the build scripts (package.json scripts section) reference jlpm directly.
Running pip show jupyterlab to verify JupyterLab was actually installed in my active environment, since jlpm is provided as a wrapper script bundled with the JupyterLab Python package, not by yarn/npm.
Finding that my JupyterLab install had gone into a different virtual environment than the one I was running commands from, so its bin/jlpm script was never on my active PATH.
Fix: activating the correct virtual environment (the one where jupyterlab was actually installed) resolved the missing jlpm command.
Bug Reproduction
Steps to Reproduce
Launch a JupyterLite notebook in the browser (e.g., via the hosted demo or a local jupyter lite serve build).
Open a notebook that includes at least one ipywidgets widget (widgets are what trigger comm channel setup, which is what makes commInfoRequest relevant here).
Wait for the kernel to start and confirm the widget renders successfully.
Close the notebook tab/panel while the kernel is still active (i.e., don't wait for it to idle out or manually shut the kernel down first).
Open the browser's developer console and observe the output during/after notebook shutdown.
Expected Behavior

The notebook should close gracefully, allowing any pending kernel communication (including the commInfoRequest) to complete before the kernel is disposed. No errors should appear in the console during shutdown.

Actual Behavior

The kernel is disposed before the pending commInfoRequest receives a response. The following uncaught error appears in the console:

Uncaught (in promise) Error: Canceled future for comm_info_request message before replies were done

This is logged as an uncaught promise rejection, not a handled/expected shutdown message — confirming this is a bug rather than intended behavior.

Root Cause Investigation

Reading the stack trace from the original issue line by line shows this is not a bug in JupyterLite's own codebase — it originates in the @jupyterlab/services and @jupyterlab/apputils packages that JupyterLite depends on and reuses for kernel/session management:

sessioncontext.js (compiled from packages/apputils/src/sessioncontext.tsx in jupyterlab/jupyterlab) — SessionContext.restart() is called, which triggers sessionManager to restart/shut down the kernel connection.
default.js (compiled from packages/services/src/kernel/default.ts) — inside KernelConnection.restart(), the private method _clearKernelState() runs. This method loops over this._futures (the map of all pending, in-flight kernel requests) and calls future.dispose() on every single one, unconditionally — regardless of whether that future has already received its final reply.
future.js (compiled from packages/services/src/kernel/future.ts) — KernelFutureHandler.dispose() is what actually throws "Canceled future for ... message before replies were done" when a future is disposed while it is still awaiting a reply.

Root cause: _clearKernelState() treats all pending futures as safe to tear down during shutdown/restart, without first checking whether a given future is still expecting a reply. The commInfoRequest future — which is naturally in-flight at close time because it's issued to set up comm targets for widgets — gets caught in that same blanket disposal sweep and throws.

This is a stronger root-cause claim than "the shutdown order is wrong," because it points to the exact function performing an unconditional operation (this._futures.forEach(future => future.dispose())) with no completion check, rather than describing symptoms.

Supporting evidence (analogous bug)

I found a very similar report in a related project: jupyter-resource-usage#172 shows the exact same stack trace shape (_clearKernelState → cancelled future, this time for kernel_info_request instead of comm_info_request), but triggered by clicking "Restart kernel and run all cells" rather than closing a notebook. This confirms the root cause is a general property of _clearKernelState()'s unconditional future-disposal loop, not something specific to comms or to the notebook-close code path — the same underlying method misbehaves whenever any kernel restart/shutdown races against any in-flight request.

Files to Investigate / Modify

Based on the stack trace and the analogous bug above, the following files and functions are directly implicated:

packages/services/src/kernel/default.ts — KernelConnection._clearKernelState(). This is where the unconditional future.dispose() loop lives. A fix would need to either (a) await/flush any futures that are expected to resolve imminently before clearing state, or (b) mark futures disposed during a normal shutdown/restart differently from futures cancelled mid-flight for other reasons, so the error isn't thrown in the former case.
packages/services/src/kernel/future.ts — KernelFutureHandler.dispose(). This is where the error message is actually thrown. Any fix to suppress/handle this gracefully during expected shutdown would need a corresponding change here (e.g., a flag passed in from _clearKernelState() indicating "this is a normal-shutdown disposal, not a genuine cancellation").
packages/apputils/src/sessioncontext.tsx — SessionContext.restart() / the dispose path invoked on notebook close. Worth checking whether shutdown could be sequenced to wait briefly for in-flight comm setup before tearing down the kernel connection, as an alternative fix location to changing _clearKernelState() itself.

Important scope note: since default.ts, future.ts, and sessioncontext.tsx all live in the upstream jupyterlab/jupyterlab repository (via the @jupyterlab/services and @jupyterlab/apputils packages that JupyterLite consumes as a dependency), the actual code fix may need to be proposed as a PR against jupyterlab/jupyterlab rather than jupyterlite/jupyterlite. Alternatively, JupyterLite could work around the issue on its own side by not issuing a commInfoRequest during teardown, or by explicitly awaiting it before triggering kernel disposal. I plan to raise this scope question with maintainers in Phase III before deciding which repository the fix belongs in.

Solution Plan (UMPIRE)
U — Understand

The reported bug occurs when a notebook is closed while a commInfoRequest is still awaiting a response from the kernel. KernelConnection._clearKernelState() in default.ts disposes all pending futures unconditionally during shutdown/restart, without checking whether they're still awaiting a reply, so the in-flight commInfoRequest future throws a "Canceled future" error via KernelFutureHandler.dispose() in future.ts.

M — Match

This resembles a classic "resource torn down while a consumer still holds a live reference" bug, common in async cleanup code. The analogous jupyter-resource-usage#172 issue confirms the same _clearKernelState() method throws for kernel_info_request under a different trigger (kernel restart from the UI), meaning the fix belongs in the shared disposal path, not in comm-specific or close-specific code.

P — Plan
Reproduce the issue locally in a built JupyterLite site with a widget-containing notebook.
Confirm via browser dev tools that the trace matches _clearKernelState() → future.dispose().
Decide whether the fix should live in _clearKernelState() (skip/await in-flight futures before disposal) or in future.ts (suppress the thrown error for shutdown-triggered disposal specifically).
Determine whether this needs to be filed/fixed upstream in jupyterlab/jupyterlab, or worked around locally in JupyterLite's own session/kernel handling code.
Implement the chosen fix on the fix-issue-155-kernel-disposal branch.
I — Implement

Implementation will occur during Phase III, once the repository scope question (JupyterLab vs. JupyterLite) is confirmed with maintainers.

R — Review

After implementing the fix, I will verify that:

the notebook closes normally with a widget-containing notebook open,
no commInfoRequest cancellation error appears in the console,
kernel shutdown still completes successfully, and
existing kernel restart functionality (the code path shared with the jupyter-resource-usage#172 case) still works correctly, i.e. the fix doesn't just suppress the error superficially.
E — Evaluate

The fix will be considered successful if the browser console no longer reports the cancellation error during notebook close, notebook and kernel-restart shutdown behavior both remain stable, and no regressions are introduced in normal notebook use.

Additional Investigation

The issue was originally reported in 2021 and remains open, indicating it has persisted across multiple releases of JupyterLite (and, since the root cause lives upstream, across multiple releases of JupyterLab as well). While investigating the development environment, I also encountered tooling changes related to the project's build system (jlpm), suggesting that parts of the development workflow have evolved over time. This reinforces the importance of understanding both the current architecture and the historical evolution of the project's kernel lifecycle before implementing a fix.
