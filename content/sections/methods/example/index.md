---
title: 'Example using httpx'
summary: 'Using httpx for own web handlers.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 36
---

## Idea

* The cache server will be changed into a JSON document cache server
* Each entry contains an ID and a raw content
* The raw content is a JSON document itself
* Here Go provides the type `json.RawMessage`

## Example

### The JSON cache server package

```go
package jsoncache

import (
    "encoding/json"
    "net/http"
    "sync"

    "./pkg/httpx"
)

// JSONDoc describes one entry in the cache server.
type JSONDoc struct {
    ID      string          `json:"id"`
    Content json.RawMessage `json:"content"`
}

// Handler provides a simple JSON in-memory cache server. Cache is 
// done via a map of string to JSONDoc. The JSONDoc contains the ID and a
// raw content. The sync.RWMutex is used to ensure that the cache is
// thread-safe.
type Handler struct {
    mu    sync.RWMutex
    cache map[string]JSONDoc
}

// NewHandler creates the cache server. It's simply needed to create
// the map of string to JSONDoc.
func NewHandler() *Handler {
    return &Handler{
        cache: make(map[string]JSONDoc),
    }
}

// ServeHTTPGet retrieves a JSON document out of cache.
func (h *Handler) ServeHTTPGet(w http.ResponseWriter, r *http.Request) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    rs := httpx.PathToResources(r, "/api/v1")
    if rs == nil {
        http.Error(w, "resource and ID are missing", http.StatusInvalidRequest)
        return
    }
    if rs[0].Name != "json-cache" {
        http.Error(w, "resource is not json-cache", http.StatusBadRequest)
        return
    }
    id := rs[0].ID
    doc, ok := h.cache[id]
    if !ok {
        http.Error(w, "JSON document not found", http.StatusNotFound)
        return
    }
    err := https.WriteBody(w, httpx.ContentTypeJSON, doc) 
    if err != nil {
        log.Printf("Error writing body: %v", err)
    }
}

// ServeHTTPPost adds a new JSON document to the cache.
func (h *Handler) ServeHTTPPost(w http.ResponseWriter, r *http.Request) {
    h.mu.Lock()
    defer h.mu.Unlock()

    rs := httpx.PathToResources(r, "/api/v1")
    if rs == nil {
        http.Error(w, "resource is missing", http.StatusInvalidRequest)
        return
    }
    if rs[0].Name != "json-cache" {
        http.Error(w, "resource is not json-cache", http.StatusBadRequest)
        return
    }
    var doc JSONDoc
    err := httpx.ReadBody(r, &doc)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    _, ok := h.cache[doc.ID]
    if ok {
        http.Error(w, "JSON document already exists", http.StatusConflict)
        return
    }
    h.cache[doc.ID] = doc
    w.WriteHeader(http.StatusCreated)
}
```

### Using the JSON cache server

```go
package main

import (
    "log"
    "net/http"

    "./pkg/httpx"
    "./pkg/jsoncache"
)

// main runs the cache server.
func main() {
    h := httpx.NewMethodHandler(jsoncache.NewHandler())
    err := http.ListenAndServe(":8080", h)

    if err != nil {
        log.Fatal(err)
    }
}
```
