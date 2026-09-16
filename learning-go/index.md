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

I'll be adding to this page as I actually run into the next thing, not before.
