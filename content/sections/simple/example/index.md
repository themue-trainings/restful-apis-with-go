---
title: 'Example with Standard Types'
summary: 'A little example showing the development and usage of an own handler.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 24
---

## Idea

* In-memory Cache Server to store named byte sequences
* HTTP Server to serve the cache

## Example

### The cache server package

```go
package cache

import (
    "net/http"
    "sync"
)

// Handler provides a simple in-memory cache server. Cache is done
// via a map of string to []byte. The sync.RWMutex is used to ensure that
// the cache is thread-safe.
type Handler struct {
    mu    sync.RWMutex
    cache map[string][]byte
}

// NewHandler creates the cache handler. It's needed to create the
// map of string to []byte.
func NewHandler() *Handler {
    return &Handler{
        cache: make(map[string][]byte),
    }
}

// ServeHTTP implements the http.Handler interface. It only handles
// two HTTP methods: GET and POST. The work itself is done by
// private helper methods.
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Map HTTP methods to individual methods.
    switch r.Method { 
    case http.MethodGet:
        h.handleGet(w, r)
    case http.MethodPost:
        h.handlePost(w, r)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
    }
}

// handleGet handles GET requests. If the path can be found in the cache,
// its data is returned. Otherwise, the response is set to 404. Only
// read lock is needed.
func (h *Handler) handleGet(w http.ResponseWriter, r *http.Request) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    // Check if path is known.
    data, ok := h.cache[r.URL.Path]
    if !ok {
        w.WriteHeader(http.StatusNotFound)
        return
    }
    w.WriteHeader(http.StatusOK)
    w.Write(data)
}

// handlePost handles POST requests. It does not matter if the path is known.
func (h *Handler) handlePost(w http.ResponseWriter, r *http.Request) {
    h.mu.Lock()
    defer h.mu.Unlock()

    // Simply write/update the variable.
    w.WriteHeader(http.StatusOK)
    h.cache[r.URL.Path] = r.Body
}
```

### Using the cache server

```go
package main

import (
    "log"
    "net/http"

    "./pkg/cache"
)

// main runs the cache server.
func main() {
    h := cache.NewHandler()
    err := http.ListenAndServe(":8080", h)

    if err != nil {
        log.Fatal(err)
    }
}
```
