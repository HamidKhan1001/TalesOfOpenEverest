---
layout: default
title: Learning Go
permalink: /learning-go/
---

<div class="hero">
  <span class="kicker">concepts, only once i've actually needed them</span>
  <h1>Learning Go</h1>
  <p>Not working through a Go book here. Each thing on this page is a concept I only actually understood because I needed it to read or write real OpenEverest code. Haven't hit it in the codebase yet, it's not on here.</p>
</div>

## interfaces as contracts, not inheritance

First Go idiom I actually got by reading real code instead of a tutorial: `internal/server/handlers/handler.go` in the OpenEverest repo.

The API server processes every request through three steps: check it's well-formed, check the user's allowed to do it, then actually talk to Kubernetes. In a language with classes I'd reach for inheritance. Go does it differently. One small `Handler` interface, three separate structs (`valhandler`, `rbachandler`, `k8shandler`) that each implement it. Every handler knows its own job and knows how to call `SetNext` to hand off to whatever's after it. None of them know the full chain, just the next one in line.

```go
type Handler interface {
    Handle(ctx echo.Context) error
    SetNext(next Handler)
}
```

This is basically what got me to stop thinking of Go interfaces as "class hierarchy, different syntax." A Go interface isn't something you inherit from, it's just a promise: this type has these methods. Any struct with `Handle` and `SetNext` satisfies `Handler`, no `implements Handler` written anywhere. Which is exactly what makes it possible to swap `valhandler` out in a test without touching the other two.

## the toolkit, before touching any code

Checked `go.mod` before writing a single line, because what a Go codebase depends on tells you what patterns you're actually about to be reading. OpenEverest's:

- **[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)**: the standard way to write a Kubernetes operator in Go. A controller watches for changes and runs a `Reconcile` function that makes reality match what the object says it should be. This is the pattern behind everything in `internal/controller/`.
- **[echo](https://echo.labstack.com/)**: the HTTP framework under the API server. Used Express or Flask before, it'll feel familiar: register routes, attach middleware, write handlers.
- **[client-go](https://github.com/kubernetes/client-go)** and **apimachinery**: the official libraries for talking to a Kubernetes API server. controller-runtime sits on top of these.
- **[casbin](https://casbin.org/)**: a policy engine for access control, separate from Kubernetes' own RBAC. Decides whether a given user can hit a given endpoint.
- **[Cobra](https://github.com/spf13/cobra)**: behind almost every serious Go CLI, `everestctl` included. Each subcommand is its own small piece registered onto a root command.

None of this is OpenEverest-specific, it's the same stuff you'd find across a big chunk of the Kubernetes ecosystem. So understanding it here isn't wasted if I ever touch a different project.

Adding to this as I actually run into the next thing, not before.
