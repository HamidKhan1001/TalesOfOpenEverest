---
title: "The PR that died in a merge conflict"
subtitle: "A fabricated timestamp, an abandoned fix that was actually right, and proving it before opening my mouth"
date: 2026-09-26
permalink: /blog/2026/09/26/the-pr-that-died-in-a-merge-conflict/
tags: [open-source, go, debugging]
pr_url: https://github.com/openeverest/provider-percona-server-mongodb/pull/107
---

Went looking for another quick fix across the org today. First lead didn't pan out, and it taught me something before the real story even started: a bug I'd flagged days ago (`events.Hub` double-close panic) turned out to already be fixed, months ago, by a completely different PR. The issue was just never closed. I'd assumed "still open" meant "still broken." Wrong assumption, caught before I acted on it, by actually reading the current source instead of trusting the issue tracker's state.

## the actual bug

`provider-percona-server-mongodb`'s `SyncBackup` and `SyncRestore` were reporting fake backup timestamps. `CompletedAt` was just `metav1.Now()`, whatever time the reconciler happened to run, not when the backup actually finished. `StartedAt` wasn't populated at all. For an audit trail, that's exactly backwards from useless, it's actively misleading.

Someone had already tried to fix this, in a PR that got closed without explanation. Before touching any code, I read why.

## reading the autopsy before writing the eulogy

The review was still there: a maintainer had confirmed the actual fix, the field mapping itself, was correct. What killed the PR was a merge conflict, resolved backward, that silently reverted an unrelated, already-merged change and took the previous author's build and tests down with it. They never noticed, went quiet, and the PR died from that instead of anything wrong with their code.

That's a completely different failure mode than "the fix was wrong," and it changes what you do next. If the fix had been wrong, I'd need to re-derive it. Since it wasn't, I could just redo the same fix on a clean, current base, where that stale merge conflict couldn't exist in the first place. Confirmed the bug was still live on `main` and that the previously-clobbered code was intact there. Then wrote it.

## proving the test isn't decorative

Wrote a unit test for the fix using a fake Kubernetes client, first time doing that. It passed. Passing isn't the same as meaning something, though, a test that passes regardless of whether the bug exists is worthless. So I broke my own fix on purpose, put the old `metav1.Now()` line back, reran the test, watched it fail with exactly the wrong value, then restored the real fix. That's the only way to actually know a test would catch a regression instead of just going through the motions.

## then the actual thing I was worried about

A passing mocked test still isn't proof something works against a real system. So I stood up the real core, attached this branch to a real cluster, created a real `Instance`, and created a real `Backup` against an actual running MongoDB. Hit a real validation error along the way (CPU limit has to be at least 600m, learned that from the provider's own logs, not a guess). Once it went through:

```
$ kubectl get backup psmdb-test-backup -o yaml
status:
  completedAt: "2026-09-26T09:02:47Z"
  startedAt: "2026-09-26T09:02:42Z"
  state: Succeeded

$ kubectl get perconaservermongodbbackup psmdb-test-backup -o jsonpath='{.status.start} {.status.completed}'
2026-09-26T09:02:42Z 2026-09-26T09:02:47Z
```

Our object's timestamps match the operator's own raw status, to the second. Not "looks plausible." Matches.

## the bonus bug, and where credit goes

Getting that live test running at all took a detour: a fresh clone of this repo can't even finish `make dev-up`, the Tiltfile never runs `helm dependency build`, so the chart is missing a required subchart on the very first deploy. Searched properly before assuming nobody had reported it (nothing came up), then filed it as its own issue with the exact fix, borrowed from a sibling provider repo that already does this correctly.

One more thing worth writing down plainly: I used Claude Code as an assistant through all of this, reading source, drafting the fix, driving the live verification. Said so directly in both the PR and the issue. Kept it to one honest sentence, no inflating what a tool did versus what I decided and reviewed myself.
