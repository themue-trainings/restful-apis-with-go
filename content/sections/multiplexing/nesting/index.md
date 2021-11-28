---
title: 'Nesting of Handlers'
summary: 'Implementing an own handler for nested handlers.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 42
---

## Requirements in RESTful APIs

* Requests like `GET /api/v1/users/1/addresses/5` show the usage of resources and subresources together with their individual identifiers
* A typical pattern is `{prefix}/{resource}/{id}/{subresource}/{subresource-id}/...`
* The type `http.ServeMux` does not handle a direct matching as IDs as the part of the path are changing
* So own path helpers and a multiplexer are needed

## HTTPX

* `NestedMux` is a multiplexer of handlers based on resource names
* A path is a number of resource names separated by slashes
* E.g. `customers`, `customers/addresses`, and `customers/contracts` are all valid paths

```go
// NestedMux allows to nest handler following the pattern
// {prefix}/{resource}/{id}/{subresource}/{subresource-id}/...
type NestedMux struct {
    mu       sync.RWMutex
    prefix   string
    handlers map[string]http.Handler
}

// NewNestedMux creates an empty nested multiplexer.
func NewNestedMux(prefix string) *NestedMux {
    return &NestedMux{
        prefix:   prefix,
        handlers: make(map[string]http.Handler),
    }
}

// Handle registers the handler for the given resource name. Nested names
// are separated by a slash.
func (mux *NestedMux) Handle(path string, h http.Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    mux.handlers[path] = h
}

// ServeHTTP implements http.Handler.
func (mux *NestedMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    ress := PathToResources(r, mux.prefix)
    path := ress.Path()
    h, exists := mux.handlers[path]

    if !exists {
        h = http.NotFoundHandler()
    }

    h.ServeHTTP(w, r)
}
```

## Links

* File [nesting.go](https://github.com/tideland/go-httpx/blob/main/nesting.go) of Tideland Go HTTPX
