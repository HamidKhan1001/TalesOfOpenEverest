---
title: "Running it for real"
subtitle: "What actually happens when you stop reading the dev setup doc and start running it"
date: 2026-09-20
permalink: /blog/2026/09/20/running-it-for-real/
tags: [open-source, debugging, contributing]
pr_url: https://github.com/openeverest/openeverest/pull/3203
---

Said I wasn't going to write troubleshooting docs from guessing. So I ran the thing.

## the setup

Installed the missing pieces (Node, pnpm), cloned `helm-charts`, pointed `dev/.env` at it, ran `make dev-up` for real against a fresh k3d cluster. First thing that broke was exactly what the doc already warned about: port 5000, owned by macOS Control Center. Confirmed it with `lsof`, changed the port, moved on. Good sign, the existing doc was right.

## the part that actually went wrong

Everything else did too, just not because of anything in `dev/README.md`.

My laptop ran out of disk. Twice. First from an aborted Homebrew build that tried to compile Node from source (this Mac is too old for Homebrew's prebuilt binaries anymore), second from something I only found by actually watching the logs: the frontend build kept rebuilding itself in an infinite loop. `fsnotify: queue or buffer overflow`, then Tilt would delete and rebuild the whole frontend again, immediately, forever. I checked why in `dev/Tiltfile`: its ignore list only excludes `dist/`, not the other temp files the build tooling writes elsewhere. Tilt was watching its own output and calling it a source change.

Docker Desktop crashed under the disk pressure. Had to restart it. Cleared 14GB with `docker system prune`. Disabled the looping resource by hand to let the real backend build finish. None of this is glamorous but all of it is real, and now it's written down in the PR as troubleshooting content instead of guessed.

## the mistake that almost shipped

Went back to double check my own commit messages before marking the PR ready, since I didn't want to put a citation in front of a reviewer that I hadn't actually verified. Good thing I did: I'd cited `dev/Tiltfile:44-46` for something that's actually at `46-48`. Off by two lines, small, but wrong is wrong.

Fixed it by amending the commit. Except my amend command dropped the DCO sign-off line entirely, I used `-m` flags without `-s`. CI caught it in about ten seconds: `DCO fail`. Fixed that too, verified the fix, force-pushed again. Small chain of mistakes, but each one caught before it went anywhere near a human reviewer.

## where it landed

[#3203](https://github.com/openeverest/openeverest/pull/3203) is marked ready for review now. Three commits: the setup-order fix, a link to the provider hub instead of a hardcoded list, and two troubleshooting entries that are in there because I watched them happen, not because I assumed they might. I also ran the full provider-install path for real, an actual PostgreSQL cluster came up through the documented dual-Tilt workflow, no changes needed there, it already works.
