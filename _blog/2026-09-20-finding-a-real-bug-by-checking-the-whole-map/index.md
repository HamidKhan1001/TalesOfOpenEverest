---
title: "Finding a real bug by checking the whole map"
subtitle: "OpenEverest isn't one repo. Turns out that matters when you're hunting for something small to fix."
date: 2026-09-20
permalink: /blog/2026/09/20/finding-a-real-bug-by-checking-the-whole-map/
tags: [open-source, go, debugging]
---

Wanted a second small thing to work on. Went looking for a "good first issue" the normal way first, label search, one repo, and came up empty again, everything real was either claimed in the comments or had an open PR sitting against it already, even when the assignee field said otherwise.

So I stopped looking in one repo and looked at the whole thing.

## openeverest is not one project

I'd been treating "OpenEverest" as the `openeverest/openeverest` repo this whole time, since that's the one with the CONTRIBUTING.md and the assign bot and the CI I've gotten used to. It's not. The org has over thirty repos: a hub for discovering plugins and providers, a shared helm-charts repo, a whole family of `provider-*` repos (Postgres, MongoDB, MySQL, ClickHouse, Milvus, and more), a handful of `plugin-*` repos, the website, a homebrew tap, governance docs. Most of them have their own issue trackers, their own CI, and, I learned the hard way, not all of them even have the `/assign` bot. Some just work on the honor system: comment that you're on it, send the PR.

Scanning all of it for real, unclaimed, small issues took a while, most of what turned up was either claimed the second I looked closer, or actually a multi-day content task wearing a "good first issue" label. But one thing stood out because it wasn't labeled easy at all: `plugin-audit`, an audit-logging plugin, had an open bug from three days earlier where its login function was calling an API endpoint that doesn't exist on `main` anymore.

## first real go fix

Every PR before this one was documentation. This was the first actual code, and it was small enough to be a good place to start: one function, `getToken()`, reading a password from config, POSTing it to the wrong URL, and parsing a token that no longer comes back in the shape it expects.

I didn't want to trust the issue's own read of what changed, so before writing anything I hit the actual running Everest server myself: the old endpoint, confirmed `404`. The new one, confirmed the real request and response shape by looking at the actual response, not the Go struct definitions. Turned out the new response includes the token's expiry directly, which let me delete a whole chunk of the old code that was manually decoding a JWT to guess at it. Removing code you're sure is dead is its own small skill, Go won't warn you about an unused *function*, only unused imports and variables, so I had to actually check nothing else called it before deleting.

Then the part I actually cared about most: I built the binary and ran it against a real, currently-live Everest server with real admin credentials. Watched it authenticate and open an event stream in the log output, not "it should work," it did work, I saw it happen.

## sitting on it

Fix is done, committed, sitting on a branch. Not opening the PR tonight, wanted to actually sleep on it and open it properly tomorrow rather than fire it off at 3am because it was ready. The rest of you reading this in order: the draft PR isn't up yet when this post goes live. That's deliberate, not forgotten.
