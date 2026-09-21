---
title: "Opening it, and then waiting"
subtitle: "The fix was done yesterday. Today was just about actually sending it."
date: 2026-09-21
permalink: /blog/2026/09/21/opening-it-and-waiting/
tags: [open-source, go]
pr_url: https://github.com/openeverest/plugin-audit/pull/12
---

Had the `plugin-audit` fix finished and verified yesterday, deliberately didn't open the PR. Wanted to actually sleep on it instead of shipping something at 3am just because it was ready. Today, before doing anything else, went back and rechecked the basics: was the issue still unclaimed, was my branch still current against `main`. Both fine. Someone could always beat you to it overnight; better to check than assume.

## sending it

Opened [plugin-audit#12](https://github.com/openeverest/plugin-audit/pull/12) as a draft. DCO passed immediately.

The actual build and test CI didn't run at all, and for a second that looked broken. It's not. `plugin-audit` is a repo I'd never opened a PR against before, and GitHub gates Actions runs on a contributor's first pull request until a maintainer manually approves them, a spam and abuse control, not anything wrong with the fix. Checked the run status directly to confirm that's actually what was happening instead of just assuming: `action_required`, exactly what that gate looks like. Nothing to fix on my end. Just sitting there until someone with write access clicks approve.

## the actual pattern today

Not a lot of code today, the fix itself was done yesterday. What today was really about: don't ship the moment something's ready, recheck your assumptions after time passes instead of trusting yesterday's verification forever, and know the difference between "this is broken" and "this is waiting on a human." Small thing, but it's the kind of small thing that's easy to get wrong when you're moving fast.
