# codepath-open-source-contribution2
# Open Source Contribution – JupyterLite Issue #155

## Project Information

**Repository:** https://github.com/jupyterlite/jupyterlite

**My Fork:** https://github.com/tiffanyelangwa/jupyterlite

**Issue:** https://github.com/jupyterlite/jupyterlite/issues/155

**Issue Title:** Kernel gets disposed before it gets a chance to respond to commInfoRequest

---

# Why I Chose This Issue

I chose JupyterLite issue #155 because it combines technologies that align well with my current programming experience and learning goals. I have experience programming in Python and have been building software projects that involve debugging, understanding existing codebases, and reasoning through application behavior. Contributing to JupyterLite will give me experience working with a mature open-source project while exposing me to how browser-based notebook environments communicate with kernels.

My primary learning goal is to become more comfortable navigating a large production codebase, reproducing reported bugs, understanding the interaction between different software components, and collaborating with maintainers throughout the open-source contribution process.

From reading the issue, I understand that when a notebook is closed, JupyterLite disposes of the kernel before it has finished responding to a `commInfoRequest`. As a result, an asynchronous request is cancelled prematurely, producing an error even though the notebook is simply shutting down. My goal is to reproduce this behavior, identify where the kernel lifecycle is being interrupted, and investigate an appropriate fix.

---

# Problem Summary

The issue occurs when a notebook is closed while the application is still waiting for the kernel to respond to a `commInfoRequest`. Instead of allowing the request to finish gracefully, the kernel is disposed too early, causing the application to throw a "Canceled future for comm_info_request" error. Although this happens during shutdown, it creates unnecessary errors and suggests that the kernel shutdown sequence is not being handled correctly. I chose this issue because it is a well-scoped bug that will help me learn about asynchronous programming, kernel lifecycle management, and debugging within a real-world open-source project.

---

# Why This Project

JupyterLite is an actively maintained open-source project with detailed documentation and a clear contribution workflow. The issue is currently open, labeled as both **good first issue** and **help wanted**, making it an appropriate contribution for a new contributor while still requiring meaningful investigation and debugging.

---

# Initial Investigation

From reading the issue discussion, the problem appears to occur during the notebook shutdown process. The kernel receives a `commInfoRequest`, but before it has the opportunity to send its response, the shutdown sequence disposes of the kernel and cancels the pending request. This results in the following error:

```
Canceled future for comm_info_request message before replies were done
```

At this stage, I have not yet reproduced the bug locally. My next step will be to set up the development environment, reproduce the behavior consistently, inspect the shutdown sequence, and identify exactly where the request is being cancelled.

---

# Likely Files / Components to Investigate

Based on the issue description and stack trace, I expect to investigate:

- The kernel lifecycle and shutdown logic
- Session context management
- Kernel future handling for asynchronous requests
- The code responsible for processing `commInfoRequest`

These components appear to be responsible for coordinating notebook shutdown and kernel communication.

---

# Acceptance Criteria

I will consider the issue successfully resolved if:

- I can reliably reproduce the reported behavior.
- I identify why the kernel is being disposed before the pending request completes.
- The shutdown sequence no longer produces the `commInfoRequest` cancellation error.
- The proposed fix does not introduce regressions in normal notebook behavior.
- Any relevant tests continue to pass after the change.

---

# Current Status

- Repository forked
- Issue selected
- README created
- Waiting to hear back after expressing interest in the issue
- Development environment setup
