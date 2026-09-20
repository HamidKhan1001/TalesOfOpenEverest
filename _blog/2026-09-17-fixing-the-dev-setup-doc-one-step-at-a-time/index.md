---
title: "Fixing the dev setup doc, one step at a time"
subtitle: "Why I'm opening this one as a draft, and a disclosure I'm adding going forward"
date: 2026-09-17
permalink: /blog/2026/09/17/fixing-the-dev-setup-doc-one-step-at-a-time/
tags: [open-source, documentation, contributing]
pr_url: https://github.com/openeverest/openeverest/pull/3203
---

Second issue, different kind of problem than the last one. [#2813](https://github.com/openeverest/openeverest/issues/2813) asked for `dev/README.md` to be fixed up for OpenEverest v2's local dev setup. Someone had already tried this once, in a PR that got closed after review feedback said the doc had gotten too sprawling to review in one sitting. Classic OpenEverest CONTRIBUTING.md advice, which I'd read by that point: keep pull requests small.

## most of the issue was already fixed

Before writing anything I read the actual issue body and checked what still needed doing. Turns out most of it didn't. Three separate PRs had already landed since the issue was filed, each fixing one specific complaint from that earlier review: dropping a stale operator reference, pointing at the right upstream repo, adding a missing prerequisite note. Nobody closed the issue when those merged so it still showed as open work, but the actual list of things wrong with the doc had shrunk a lot without me touching anything.

Good reminder before picking up any "still open" issue: go check what's actually still true, not just what the issue says.

## the bug that was still there

One real thing left, and I only trust it because I checked it against the code instead of assuming. Doc's setup section told you to run `make dev-up` first, then only afterward walked you through setting `EVEREST_CHART_DIR` and the rest of the Tilt config.

`make dev-up` in the `Makefile` creates the cluster and starts Tilt in the same command. Tilt reads `EVEREST_CHART_DIR` the second it starts and fails outright if it's missing. So doing the steps in the order the doc gave you crashes on the very first command. Fix was simple once I'd actually confirmed that: move the config steps ahead of the command that needs them, drop the now-duplicate "start Tilt" step that showed up again later.

## draft, and a disclosure

Two calls on this one, both from stuff I read in `CONTRIBUTING.md` that I hadn't paid close enough attention to the first time.

First: opened the PR as a draft. Guidelines say to do that while you're still working, mark it ready when you actually want eyes on it. I'm planning to work through the rest of #2813 (provider repo links, more troubleshooting cases, a note about the `release-2.0` rename) in separate small pieces instead of one big PR, so draft is just the honest state for it right now.

Second, and this is the one worth actually explaining instead of burying: `CONTRIBUTING.md` has a section asking contributors to say when a contribution is substantially AI-assisted. I use Claude Code as part of how I work, mostly to dig through a codebase faster than I could alone and to double check my reasoning against the actual source instead of my assumptions. That's literally what happened with the `make dev-up` bug above. So the PR description just says so. Small line, costs nothing, matches what the project's actually asking for.

PR is up: [#3203](https://github.com/openeverest/openeverest/pull/3203). Not ready for review yet, still in progress.
