---
layout: default
title: About OpenEverest
permalink: /about/
---

<div class="hero">
  <span class="kicker">what it is, before any of the blog posts make sense</span>
  <h1>About OpenEverest</h1>
  <p>Before the PR stories here make sense, you kind of need to know what OpenEverest actually is. So here's the plain version, from reading its README and source, not the marketing page.</p>
</div>

<div class="callout">
Quick heads up: OpenEverest currently has two live versions. <code>main</code> is <strong>v2</strong>, a developer preview, not what most people run in prod yet. The actual production release is <strong>v1</strong>, on branch <code>v1.x</code>. Everything here is about v2 unless I say otherwise, because that's the branch I'm working against.
</div>

## the problem it's solving

Running a database properly on Kubernetes is more work than most teams want to do by hand: provisioning, backups, restores, monitoring, upgrades, who's allowed to touch what. OpenEverest is an open source platform that installs into a cluster and takes that off your plate. You describe what you want, a database, a backup schedule, some permissions, and its controllers make the cluster match that and keep it that way.

You never touch Kubernetes directly. Three doors, same destination:

- the web UI, if you want to click through it
- `everestctl`, the CLI, for scripting
- the REST API directly, if you're building your own thing on top

All three hit the same API server, which is the only thing actually talking to Kubernetes for you.

## the data model, aka what actually gets created

Here's the part that took me the longest to get: OpenEverest doesn't really have "features." It has Kubernetes custom resources, and controllers that react when they change. Whatever you do through the UI or CLI eventually becomes one of these.

<div class="diagram">
<svg viewBox="0 0 680 380" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Diagram of OpenEverest's core custom resources and how they relate: Instance, Provider, Backup, Restore, BackupStorage, MonitoringConfig, Plugin, InstalledExtension">
  <defs>
    <marker id="arrow3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>
  <g font-family="IBM Plex Mono, monospace">
    <rect x="255" y="15" width="170" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="340" y="38" text-anchor="middle" font-size="13" font-weight="600">Instance</text>
    <text x="340" y="55" text-anchor="middle" font-size="10.5" class="muted">the database you asked for</text>

    <rect x="255" y="105" width="170" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="340" y="128" text-anchor="middle" font-size="13" font-weight="600">Provider</text>
    <text x="340" y="145" text-anchor="middle" font-size="10.5" class="muted">which database engine runs it</text>

    <rect x="30" y="195" width="150" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="105" y="218" text-anchor="middle" font-size="13" font-weight="600">Backup</text>
    <text x="105" y="235" text-anchor="middle" font-size="10.5" class="muted">one backup run</text>

    <rect x="200" y="195" width="150" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="275" y="218" text-anchor="middle" font-size="13" font-weight="600">Restore</text>
    <text x="275" y="235" text-anchor="middle" font-size="10.5" class="muted">restoring from one</text>

    <rect x="115" y="285" width="170" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="200" y="308" text-anchor="middle" font-size="13" font-weight="600">BackupStorage</text>
    <text x="200" y="325" text-anchor="middle" font-size="10.5" class="muted">where backups live, e.g. S3</text>

    <rect x="380" y="195" width="150" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="455" y="218" text-anchor="middle" font-size="13" font-weight="600">MonitoringConfig</text>
    <text x="455" y="235" text-anchor="middle" font-size="10.5" class="muted">watches the instance</text>

    <rect x="530" y="15" width="150" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="605" y="38" text-anchor="middle" font-size="13" font-weight="600">Plugin</text>
    <text x="605" y="55" text-anchor="middle" font-size="10.5" class="muted">an installable add-on</text>

    <rect x="530" y="105" width="150" height="55" rx="2" fill="none" stroke="#A8431E" stroke-width="1.3"/>
    <text x="605" y="128" text-anchor="middle" font-size="13" font-weight="600">InstalledExtension</text>
    <text x="605" y="145" text-anchor="middle" font-size="10.5" class="muted">tracks install/upgrade of a Plugin or Provider</text>
  </g>
  <g style="color:#A8431E" stroke="currentColor" stroke-width="1.2" fill="none">
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
<p class="diagram-caption">simplified. an Instance runs on a Provider. Backups and Restores point at a BackupStorage. MonitoringConfig watches an Instance. Plugins and Providers both get installed through InstalledExtension (dashed line: yeah, it can install a Provider too, not just a Plugin, naming is a little confusing).</p>
</div>

If you know v1, the big change in v2 is right at the top of that diagram. v1 had one resource, `DatabaseCluster`, that assumed a small fixed list of supported databases. v2 swapped that for the generic `Instance` plus a pluggable `Provider`, so new engines can get added without touching core.

## the plugin system

Honestly the part I find most interesting, and it's newer than the rest of this. OpenEverest lets you extend it without touching core code, on both ends:

- **backend**: a `Plugin` resource declares a frontend bundle, an optional backend service, and the permissions it needs. The API server exposes `GET /v1/plugins` for discovery and proxies `/v1/plugins/:name/*` straight to that plugin's backend, re-reading the `Plugin` object from Kubernetes on every request. So a change takes effect without restarting anything.
- **frontend**: the web UI loads each plugin's JS bundle and lets it register into specific slots, a route, a sidebar item, a tab, a settings panel, a dashboard widget. It doesn't get free rein over the UI, just named places to plug into.

Basically the same idea Kubernetes uses for CRDs, one layer up: don't hardcode every feature, give people a defined way to bolt their own on.

## what it's built with

- **backend**: Go 1.26. API server runs on [echo](https://echo.labstack.com/). Controllers use [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime), the standard for writing Kubernetes operators. Permissions go through [casbin](https://casbin.org/). CLI is built with [Cobra](https://github.com/spf13/cobra).
- **frontend**: React, TypeScript, Vite, MUI for components, TanStack Query for data.
- **packaging**: installs via Helm.

More internals, with diagrams of how a request actually moves through the system, on the [Architecture](/TalesOfOpenEverest/architecture/) page.

*written from reading /README.md, go.mod, api/, internal/server/plugins.go, and docs/process/generic-plugins-design.md in the repo. i'll fix this page as i understand more instead of leaving a wrong first draft up.*
