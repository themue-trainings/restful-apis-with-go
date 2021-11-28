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

const (
    // Prefix for API calls.
    apiPrefix = apiPrefix

    // Name of the jsoncache resource.
    resourceName = "jsoncache"
)

// jsonDoc describes one entry in the cache server. It's a wrapper
// containing the ID and the raw JSON document. It's the content of
// the map.
type jsonDoc struct {
    ID      string          `json:"id"`
    Content json.RawMessage `json:"content"`
}

// Handler provides a simple JSON in-memory cache server. Cache is 
// done via a map of string to jsonDoc. The jsonDoc contains the ID and a
// raw content. The sync.RWMutex is used to ensure that the cache is
// thread-safe.
type Handler struct {
    mu    sync.RWMutex
    cache map[string]jsonDoc
}

// NewHandler creates the cache server. It's simply needed to create
// the map of string to jsonDoc.
func NewHandler() *Handler {
    return &Handler{
        cache: make(map[string]jsonDoc),
    }
}

// ServeHTTPGet retrieves a JSON document out of cache.
func (h *Handler) ServeHTTPGet(w http.ResponseWriter, r *http.Request) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    ress := httpx.PathToResources(r, apiPrefix)
    if ress == nil {
        http.Error(w, "resource and ID are missing", http.StatusInvalidRequest)
        return
    }
    if !ress.IsPath(resourceName) {
        http.Error(w, "resource is not json-cache", http.StatusBadRequest)
        return
    }
    doc, ok := h.cache[ress.PathID(resourceName)]
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

    if !httpx.PathToResources(apiPrefix).IsPath(resourceName) {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    var doc jsonDoc
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
    h := jsoncache.NewHandler()
    mh := httpx.NewMethodHandler(h)
    err := http.ListenAndServe(":8080", mh)

    if err != nil {
        log.Fatal(err)
    }
}
```
