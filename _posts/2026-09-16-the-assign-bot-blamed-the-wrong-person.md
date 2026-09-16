---
title: "The assign bot was blaming the wrong person"
subtitle: "My first real review cycle on OpenEverest, and the git mistake that almost held it up"
date: 2026-09-16
tags: [open-source, github-actions, code-review]
pr_url: https://github.com/openeverest/openeverest/pull/3118
---

OpenEverest has a small GitHub Action bot that handles issue assignment. Comment `/assign` on an issue and it assigns you, checks you're not already holding too many open issues, and replies with what to do next. Simple enough, and it works fine for the normal case: someone claiming their own issue.

It breaks down for the other case. A maintainer can delegate an issue by commenting `/assign @someone`. If GitHub rejects that assignment, which it does if the target hasn't commented on the issue yet, the bot's failure message still said: *"I couldn't assign this to you, a maintainer will pick this up shortly."* Read that again as the maintainer who just typed the command. You are the maintainer. Nobody is coming to pick it up. That's issue [#2839](https://github.com/openeverest/openeverest/issues/2839).

## Redoing someone else's stalled attempt

Someone had already tried to fix this in [#2841](https://github.com/openeverest/openeverest/pull/2841). It stalled for reasons that had nothing to do with the fix itself: the branch targeted the old `v1.x` line instead of `main`, so CI couldn't even run, and there was a DCO problem on top of that. I redid it fresh against `main` instead of trying to rescue that branch.

My first version branched the message on whether the person who commented was the same person being assigned, and named the likely cause when they weren't: the target probably hadn't commented on the issue yet, which is GitHub's actual eligibility rule for assignees.

## What review actually caught

Two things came back in review that I hadn't thought of on my own.

First, from [julismo](https://github.com/openeverest/openeverest/pull/3118#issuecomment-5568700139): my fix still pinged the *commenter*, the maintainer, not the *target*, the person who was supposed to get the issue. If you're the target, you'd never even see a message addressed to someone else.

Second, and this is the one I actually like: my proposed recovery path was "ask the target to comment here, then ask the maintainer to retry `/assign`." Two people, two steps. But the target can just comment `/assign` themselves. That one comment both satisfies GitHub's "has participated in the thread" requirement and runs the self-assign flow in the same action. One step instead of two, and it puts the fix in the hands of the person who actually wants the issue instead of routing back through the maintainer.

[recharte](https://github.com/openeverest/openeverest/pull/3118#pullrequestreview-5207061635) asked for exactly that change before merging. So the message now reads:

> `@target, @commenter tried to assign this issue to you, but GitHub didn't apply the assignment. This can happen if you aren't a collaborator and haven't commented on this issue yet. To try taking it yourself, post /assign on its own line in a new comment here.`

## The part that actually slowed me down

Not the message logic. Git config.

I amended a commit locally, and the DCO check turned red on the PR. Turned out my global `git config user.name` had been stuck on a literal placeholder string since who knows when, something I'd clearly copy-pasted from a setup guide once and never actually filled in. The amended commit inherited that placeholder as the author name, with no `Signed-off-by` line to match it, and DCO didn't approve of a commit that couldn't be traced to an actual person.

Fixed the config, amended again with a proper `Signed-off-by`, force-pushed the one commit that needed it. DCO went green. It was a five-minute fix once I found it, and a reminder that the part of a PR that trips you up is rarely the part you were worried about.

`recharte` merged it not long after. First merged PR on this project. On to the next issue.
