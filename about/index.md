---
layout: default
title: About OpenEverest
permalink: /about/
---

<div class="hero">
  <p class="kicker">What it is, before I get into how I'm contributing to it</p>
  <h1>About OpenEverest</h1>
  <p>Before any of the PR stories on this site make sense, it helps to know what OpenEverest actually is. This page is that: a plain explanation of the project, written from reading its own README and source, not marketing copy.</p>
</div>

<div class="callout">
Quick note up front: OpenEverest currently has two live versions. The <code>main</code> branch is <strong>v2</strong>, a developer preview, not yet what most people run in production. The production release is <strong>v1</strong>, on branch <code>v1.x</code>. Everything on this site, unless I say otherwise, is about v2, because that's the branch I'm actually working against.
</div>

## The problem it's solving

Running a database well on Kubernetes is more work than most teams want to do by hand: provisioning, backups, restores, monitoring, upgrades, access control. OpenEverest is an open source platform that installs into a Kubernetes cluster and takes that work off your plate. You describe what you want (a database instance, a backup schedule, who's allowed to touch what), and OpenEverest's controllers make the cluster match that description and keep it that way.

You never talk to Kubernetes directly. You go through one of three doors:

- **The web UI**, built in React, for people who want to click through it
- **`everestctl`**, the CLI, for scripting and automation
- **The REST API** directly, if you're building your own tooling on top

All three doors lead to the same API server, which is the only thing that talks to Kubernetes on your behalf.

## The data model: what gets created, not what gets clicked

The part that took me the longest to actually get was this: OpenEverest doesn't really have "features" in the traditional sense. It has Kubernetes custom resources (CRDs), and controllers that react to them. Everything you do through the UI or CLI ultimately becomes one of these objects.

<div class="diagram">
<svg viewBox="0 0 680 380" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Diagram of OpenEverest's core custom resources and how they relate: Instance, Provider, Backup, Restore, BackupStorage, MonitoringConfig, Plugin, InstalledExtension">
  <defs>
    <marker id="arrow3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>
  <g font-family="Inter, sans-serif">
    <rect x="255" y="15" width="170" height="55" rx="8" fill="#e5efe9" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="340" y="38" text-anchor="middle" font-size="13" font-weight="600">Instance</text>
    <text x="340" y="55" text-anchor="middle" font-size="10.5" class="muted">the database you asked for</text>

    <rect x="255" y="105" width="170" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="340" y="128" text-anchor="middle" font-size="13" font-weight="600">Provider</text>
    <text x="340" y="145" text-anchor="middle" font-size="10.5" class="muted">which database engine runs it</text>

    <rect x="30" y="195" width="150" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="105" y="218" text-anchor="middle" font-size="13" font-weight="600">Backup</text>
    <text x="105" y="235" text-anchor="middle" font-size="10.5" class="muted">one backup run</text>

    <rect x="200" y="195" width="150" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="275" y="218" text-anchor="middle" font-size="13" font-weight="600">Restore</text>
    <text x="275" y="235" text-anchor="middle" font-size="10.5" class="muted">restoring from one</text>

    <rect x="115" y="285" width="170" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="200" y="308" text-anchor="middle" font-size="13" font-weight="600">BackupStorage</text>
    <text x="200" y="325" text-anchor="middle" font-size="10.5" class="muted">where backups live, e.g. S3</text>

    <rect x="380" y="195" width="150" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="455" y="218" text-anchor="middle" font-size="13" font-weight="600">MonitoringConfig</text>
    <text x="455" y="235" text-anchor="middle" font-size="10.5" class="muted">watches the instance</text>

    <rect x="530" y="15" width="150" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="605" y="38" text-anchor="middle" font-size="13" font-weight="600">Plugin</text>
    <text x="605" y="55" text-anchor="middle" font-size="10.5" class="muted">an installable add-on</text>

    <rect x="530" y="105" width="150" height="55" rx="8" fill="none" stroke="#1f5c4a" stroke-width="1.5"/>
    <text x="605" y="128" text-anchor="middle" font-size="13" font-weight="600">InstalledExtension</text>
    <text x="605" y="145" text-anchor="middle" font-size="10.5" class="muted">tracks install/upgrade of a Plugin or Provider</text>
  </g>
  <g style="color:#1f5c4a" stroke="currentColor" stroke-width="1.4" fill="none">
    <line x1="340" y1="70" x2="340" y2="105" marker-end="url(#arrow3)"/>
    <line x1="300" y1="160" x2="150" y2="195" marker-end="url(#arrow3)"/>
    <line x1="320" y1="160" x2="290" y2="195" marker-end="url(#arrow3)"/>
    <line x1="150" y1="250" x2="190" y2="285" marker-end="url(#arrow3)"/>
    <line x1="270" y1="250" x2="220" y2="285" marker-end="url(#arrow3)"/>
    <line x1="380" y1="160" x2="440" y2="195" marker-end="url(#arrow3)"/>
    <line x1="605" y1="160" x2="605" y2="195" marker-end="url(#arrow3)" stroke-dasharray="3 3"/>
    <line x1="605" y1="70" x2="605" y2="105" marker-end="url(#arrow3)"/>
  </g>
</svg>
<p class="diagram-caption">Simplified. An Instance runs on a Provider. Backups and Restores reference a BackupStorage. MonitoringConfig watches an Instance. Plugins and Providers themselves get installed and tracked through InstalledExtension. Dashed line: an InstalledExtension can install a Provider too, not just a Plugin.</p>
</div>

If you're used to v1, the biggest change in v2 is right at the top of that diagram: v1 had one resource called `DatabaseCluster` that assumed a small, fixed list of supported databases. v2 replaced it with the generic `Instance` plus a pluggable `Provider`, so new database engines can be added without changing the core.

## The plugin system

This is the part I found most interesting, and it's newer than most of the rest. OpenEverest lets you extend it without touching core code, on both ends:

- **Backend**: a `Plugin` resource declares a frontend bundle, an optional backend service, and the permissions it needs. The API server exposes `GET /v1/plugins` for discovery and reverse-proxies requests under `/v1/plugins/:name/*` straight to that plugin's backend, re-reading the `Plugin` object from Kubernetes on every request, so a change to a plugin takes effect without restarting anything.
- **Frontend**: the web UI loads each plugin's JS bundle and lets it register into specific extension points: a route, a sidebar item, a tab on a cluster's detail page, a settings panel, an action, a dashboard widget. The plugin doesn't get free rein over the UI, it gets specific, named slots to plug into.

It's the same idea Kubernetes itself uses for CRDs, applied one layer up: don't hardcode every feature, give people a defined way to add their own.

## What it's built with

- **Backend**: Go, `1.26`. The API server runs on the [echo](https://echo.labstack.com/) framework. Controllers are built on [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime), the standard library for writing Kubernetes operators. Permissions go through [casbin](https://casbin.org/), a policy-based access control engine. The CLI is built with [Cobra](https://github.com/spf13/cobra).
- **Frontend**: React with TypeScript, built with Vite, using MUI for components and TanStack Query for data fetching.
- **Packaging**: it installs via Helm.

More on the internals, with diagrams of how a request actually flows through the system, is on the [Architecture](/TalesOfOpenEverest/architecture/) page.

*Written from reading `/README.md`, `go.mod`, `api/`, `internal/server/plugins.go`, and `docs/process/generic-plugins-design.md` in the OpenEverest repo. I'll correct this page as I understand more, rather than leave a wrong first draft standing.*
