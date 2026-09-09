# PyTorch Elastic rendezvous: preserving socket errors

**Date:** 2026-09-09  
**Upstream issue:** [pytorch/pytorch#191395](https://github.com/pytorch/pytorch/issues/191395)  
**Pull request:** [pytorch/pytorch#196497](https://github.com/pytorch/pytorch/pull/196497)  
**Status:** Open; the contributor license agreement is signed. PyTorch CI awaits maintainer approval for a first-time contributor workflow.

## Context

I wanted to make a focused contribution to a large production codebase. I first scoped another issue, then found an existing upstream pull request and chose not to duplicate it. I selected issue #191395 because it described an isolated failure path in PyTorch Elastic's etcd rendezvous server.

## Failure trace

`find_free_port` resolves candidate addresses and creates a socket for each address. Before this change, its `except OSError` block unconditionally called `s.close()`.

If `socket.socket(...)` raised before assigning `s`, Python raised `UnboundLocalError` in the error handler. That hid the original socket failure and prevented the function from trying later resolved addresses.

## Change

I reset `s` to `None` for each address attempt and close it only when creation succeeded. The function now preserves the original `OSError`, continues to the next address when possible, and raises its intended `RuntimeError` when all attempts fail.

The change touches two files:

- `torch/distributed/elastic/rendezvous/etcd_server.py`
- `test/distributed/elastic/rendezvous/etcd_server_test.py`

## Regression coverage

The tests cover two behavior paths:

1. The first socket-construction attempt raises `OSError`; the next resolved address returns a usable socket.
2. Every socket-construction attempt raises `OSError`; the function raises `RuntimeError("Failed to create a socket")` rather than `UnboundLocalError`.

## Verification record

Completed locally:

- `python3 -m compileall -q torch/distributed/elastic/rendezvous/etcd_server.py test/distributed/elastic/rendezvous/etcd_server_test.py`
- Focused mocked checks for the retry and all-fail paths.

Not completed locally:

- The full PyTorch test suite. The shallow source checkout did not contain a built PyTorch extension, so importing the full package failed before the test module could run.

Pending externally:

- PyTorch CI. The workflow requires a maintainer to approve first-time contributor runs.

## What I learned

- Error-handling code needs the same scrutiny as the happy path. A cleanup line can hide the exception that explains the real failure.
- A narrow patch with a direct regression test is easier to review than a speculative refactor.
- Open-source contribution work begins before the final commit: issue selection, repository conventions, contributor agreements, and honest reporting all matter.
- When local infrastructure prevents a full test run, record the limitation and test the smallest reliable surface instead of claiming coverage that did not run.

## Next action

Monitor the upstream pull request, respond to reviewer feedback, and report the CI result only after GitHub publishes it.
