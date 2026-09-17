---
layout: default
title: Architecture
permalink: /architecture/
---

<div class="hero">
  <span class="kicker">written from reading the code, i'll fix it as i understand more</span>
  <h1>How OpenEverest is actually built</h1>
  <p>My own map of the codebase, not a copy of the official docs. Writing it as I understand each piece, so parts of this will get corrected later once I go deeper. If something's wrong here, that's kind of the point of a learning log.</p>
</div>

<div class="callout">
The <code>main</code> branch is the <strong>v2 developer preview</strong>, not what most people run. The production release is on <code>v1.x</code>, and v2 changed some of the core design (v1's <code>DatabaseCluster</code> resource became a more general <code>Instance</code> + pluggable <code>Provider</code> in v2). Everything below is about v2.
</div>

## what it does, in one paragraph

OpenEverest installs into a Kubernetes cluster and gives you a web UI, a REST API, and a CLI (<code>everestctl</code>) to provision and manage databases on it: creating instances, running backups and restores, watching monitoring. Built the way most serious Kubernetes platforms are: you describe what you want as a Kubernetes object, and a controller running in the background makes the cluster match that.

## the shape of the repo

| Directory | What lives there |
|---|---|
| `cmd/` | The three actual Go binaries: the API server, the controller manager, and the CLI |
| `commands/` | Every `everestctl` subcommand (backup, restore, instance, auth, and so on), built with Cobra |
| `internal/server/` | The HTTP API, built on the echo framework |
| `internal/controller/` | The Kubernetes controllers that do the real work |
| `internal/webhook/` | Admission webhooks |
| `pkg/rbac/` | The permissions engine (casbin) |
| `api/` | The Go type definitions for every custom Kubernetes resource (CRD) |
| `ui/apps/everest/` | The web app: React, TypeScript, Vite, MUI |
| `docs/` | Project docs, including a genuinely good set of UI architecture docs under `docs/ui/architecture/` |

## how a request turns into a running database

<div class="diagram">
<svg viewBox="0 0 640 470" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Flow diagram: CLI or Web UI to API Server to Kubernetes API to Controller Manager to real cluster resources">
  <defs>
    <marker id="arrow1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>
  <g style="color:#A8431E" stroke="currentColor" stroke-width="1.6" fill="none">
    <line x1="150" y1="70" x2="150" y2="100" marker-end="url(#arrow1)"/>
    <line x1="490" y1="70" x2="490" y2="100" marker-end="url(#arrow1)"/>
    <line x1="320" y1="160" x2="320" y2="190" marker-end="url(#arrow1)"/>
    <line x1="320" y1="250" x2="320" y2="280" marker-end="url(#arrow1)"/>
    <line x1="320" y1="340" x2="320" y2="370" marker-end="url(#arrow1)"/>
  </g>
  <g font-family="IBM Plex Mono, monospace">
    <rect x="60" y="10" width="180" height="60" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="150" y="35" text-anchor="middle" font-size="13" font-weight="600">everestctl</text>
    <text x="150" y="53" text-anchor="middle" font-size="11" class="muted">CLI (Cobra)</text>

    <rect x="400" y="10" width="180" height="60" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="490" y="35" text-anchor="middle" font-size="13" font-weight="600">Web UI</text>
    <text x="490" y="53" text-anchor="middle" font-size="11" class="muted">React + Vite + MUI</text>

    <rect x="140" y="100" width="360" height="60" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="320" y="124" text-anchor="middle" font-size="13" font-weight="600">API Server</text>
    <text x="320" y="142" text-anchor="middle" font-size="11" class="muted">echo · JWT auth · casbin RBAC · request validation</text>

    <rect x="140" y="190" width="360" height="60" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="320" y="214" text-anchor="middle" font-size="13" font-weight="600">Kubernetes API server</text>
    <text x="320" y="232" text-anchor="middle" font-size="11" class="muted">the cluster's source of truth</text>

    <rect x="140" y="280" width="360" height="60" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="320" y="304" text-anchor="middle" font-size="13" font-weight="600">Controller Manager</text>
    <text x="320" y="322" text-anchor="middle" font-size="11" class="muted">controller-runtime · watches CRDs, reconciles state</text>

    <rect x="90" y="370" width="460" height="80" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5" stroke-dasharray="4 3"/>
    <text x="320" y="398" text-anchor="middle" font-size="13" font-weight="600">Real cluster resources</text>
    <text x="320" y="418" text-anchor="middle" font-size="11" class="muted">Jobs that run backups/restores, Pods for the database itself,</text>
    <text x="320" y="434" text-anchor="middle" font-size="11" class="muted">Secrets, RBAC objects created on your behalf</text>
  </g>
</svg>
<p class="diagram-caption">you never talk to kubernetes directly. you go through the cli or ui, the api server checks who you are and validates the request, kubernetes stores that as the desired state, and a controller in the background makes it real.</p>
</div>

controllers i've actually read so far, in `internal/controller/`:

- **BackupReconciler**: runs an on-demand backup by creating a Kubernetes Job plus the RBAC it needs.
- **RestoreReconciler**: same idea in reverse, restoring from a `Backup` or a point-in-time source.
- **BackupStorageReconciler**: manages where backups get stored (an S3-compatible target, say).
- **MonitoringConfigReconciler**: wires up PMM monitoring for an instance.
- **PluginReconciler** and **InstalledExtensionReconciler**: install and track plugins. how OpenEverest adds functionality without baking it into core.

## the part that's actually well designed: the api handler chain

Every API request goes through three handlers, each one implementing the same tiny interface (`SetNext`, then handle-or-pass-along), chained: validate the request shape, check permissions, then actually talk to Kubernetes.

<div class="diagram">
<svg viewBox="0 0 640 140" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Handler chain diagram: validation to RBAC to Kubernetes handler">
  <defs>
    <marker id="arrow2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>
  <g style="color:#A8431E" stroke="currentColor" stroke-width="1.6" fill="none">
    <line x1="10" y1="70" x2="45" y2="70" marker-end="url(#arrow2)"/>
    <line x1="215" y1="70" x2="250" y2="70" marker-end="url(#arrow2)"/>
    <line x1="420" y1="70" x2="455" y2="70" marker-end="url(#arrow2)"/>
    <line x1="625" y1="70" x2="630" y2="70" marker-end="url(#arrow2)"/>
  </g>
  <g font-family="IBM Plex Mono, monospace">
    <rect x="45" y="35" width="170" height="70" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="130" y="65" text-anchor="middle" font-size="13" font-weight="600">Validation</text>
    <text x="130" y="83" text-anchor="middle" font-size="11" class="muted">is this request well-formed?</text>

    <rect x="250" y="35" width="170" height="70" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="335" y="65" text-anchor="middle" font-size="13" font-weight="600">RBAC (casbin)</text>
    <text x="335" y="83" text-anchor="middle" font-size="11" class="muted">is this user allowed to do it?</text>

    <rect x="455" y="35" width="170" height="70" rx="8" fill="none" stroke="#A8431E" stroke-width="1.5"/>
    <text x="540" y="65" text-anchor="middle" font-size="13" font-weight="600">Kubernetes handler</text>
    <text x="540" y="83" text-anchor="middle" font-size="11" class="muted">actually talks to the cluster</text>
  </g>
</svg>
<p class="diagram-caption">internal/server/handlers/: validation/, rbac/, k8s/. each one only knows about the next one in line, not the whole chain.</p>
</div>

Clean, textbook chain-of-responsibility. Honestly the easiest part of the server to understand, exactly because each piece only does one job and doesn't care what happens after.

## files worth reading if you're new here

- `cmd/cli/main.go`: about as small and plain as a Go `main()` gets.
- `internal/server/handlers/handler.go`: the interface behind the diagram above.
- `internal/controller/backup/backupstorage_controller.go`: the simplest controller, good first reconciler to read.
- `docs/ui/architecture/`: the maintainers already wrote solid frontend architecture docs with diagrams. read those directly instead of trusting me to re-explain it.

*sources: read directly from /README.md, go.mod, cmd/, internal/server/, internal/controller/, api/, and docs/ui/architecture/ in the repo. nothing here was run, only read, so if the code's moved on since, trust the repo over this page.*
