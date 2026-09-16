---
layout: default
title: Learning Go
permalink: /learning-go/
---

<div class="hero">
  <p class="kicker">Concepts, only once I've actually needed them</p>
  <h1>Learning Go</h1>
  <p>I'm not working through a Go book here. Each entry on this page is a concept I only understood because I needed it to read or write real OpenEverest code. If I haven't hit it in the codebase yet, it's not on here.</p>
</div>

## Interfaces as contracts, not inheritance

The first Go idiom I actually understood by reading real code, not a tutorial, was in `internal/server/handlers/handler.go` in the OpenEverest repo.

The API server processes every request through three steps: check the request is well-formed, check the user is allowed to do it, then actually talk to Kubernetes. In a language with classes, I'd reach for inheritance. Go does it differently: there's one small `Handler` interface, and three separate structs (`valhandler`, `rbachandler`, `k8shandler`) that each implement it. Every handler knows how to do its own job and how to call `SetNext` to hand off to whatever comes after it. None of them know about the full chain, only the one next in line.

```go
type Handler interface {
    Handle(ctx echo.Context) error
    SetNext(next Handler)
}
```

Roughly, this is what let me stop thinking of Go interfaces as "the class hierarchy, but different syntax." A Go interface isn't a type you inherit from. It's just a promise: "this type has these methods." Any struct that happens to have `Handle` and `SetNext` satisfies `Handler`, with no explicit `implements Handler` anywhere. That's what makes it possible to swap `valhandler` for something else in tests without touching the other two.

## The toolkit, before touching any of the code

I checked `go.mod` before writing a single line, because knowing what a Go codebase depends on tells you what patterns you're actually going to be reading. OpenEverest's are:

- **[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)**: the standard way to write a Kubernetes operator in Go. A controller watches for changes to an object and runs a `Reconcile` function that makes the real world match what the object says it should be. This is the pattern behind every file in `internal/controller/`.
- **[echo](https://echo.labstack.com/)**: the HTTP framework behind the API server. If you've used Express in Node or Flask in Python, the shape is familiar: register routes, attach middleware, write handlers.
- **[client-go](https://github.com/kubernetes/client-go)** and **apimachinery**: the official Go libraries for talking to a Kubernetes API server. controller-runtime is built on top of these.
- **[casbin](https://casbin.org/)**: a policy engine for access control, separate from Kubernetes' own RBAC. This is what decides whether a given user can call a given API endpoint.
- **[Cobra](https://github.com/spf13/cobra)**: the library behind almost every serious Go CLI, including `everestctl`. Each subcommand is its own small piece, registered onto a root command.

None of these are OpenEverest-specific. They're the same libraries you'd find in a large chunk of the Kubernetes ecosystem, so time spent understanding them here transfers directly to other projects.

I'll be adding to this page as I actually run into the next thing, not before.
