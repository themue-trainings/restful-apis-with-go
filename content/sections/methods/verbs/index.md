---
title: 'HTTP Methods are Verbs'
summary: 'Introduction into HTTP methods and how to more easy handle them.'
date: 2021-11-26T12:00:00+1:00
draft: false
weight: 31
---

## Handling HTTP methods

* HTTP methods are verbs, and are used to indicate the type of action that the client is requesting
* In RESTful APIs they are are used to indicate the action to be performed on a resource
* So a typical task of the code is to analyse the method and decide what to do
* We've seen how to use a `switch` statement to check the method and act accordingly

```go
switch r.Method {
case http.MethodGet:
    // Handle GET requests in a private method.
    h.handleGet(w, r)
case http.MethodPost:
    // Handle POST requests in a private method.
    h.handlePost(w, r)
default:
    // Handle other methods or return error.
    ...
}
```

## HTTPX

### But why repeat this for every handler?

* Let's create our HTTP helper package, named e.g. `pkg/httpx`

```go
// Package httpx contains helper functions for the daily work with HTTP.
package httpx
```

* Use the power of interfaces to distribute HTTP methods
* Create our method wrapper in `pkg/httpx/methods.go`
* It defines individual interfaces for each HTTP method
* The wrapper implements the `http.Handler` interface and contains the handler with the business logic
* Its `ServeHTTP` distributes the requests to the handler methods based on a type switch

```go
package httpx

import (
    "net/http"
)

// GetHandler has to be implemented by a handler for GET requests
// dispatched through the MethodHandler.
type GetHandler interface {
    ServeHTTPGet(w http.ResponseWriter, r *http.Request)
}

// PostHandler has to be implemented by a handler for POST requests
// dispatched through the MethodHandler.
type PostHandler interface {
    ServeHTTPPost(w http.ResponseWriter, r *http.Request)
}

// PutHandler has to be implemented by a handler for PUT requests
// dispatched through the MethodHandler.
type PutHandler interface {
    ServeHTTPPut(w http.ResponseWriter, r *http.Request)
}

// DeleteHandler has to be implemented by a handler for DELETE requests
// dispatched through the MethodHandler.
type DeleteHandler interface {
    ServeHTTPDelete(w http.ResponseWriter, r *http.Request)
}

// ...

// MethodHandler wraps a http.Handler implementing also individual httpx handler
// interfaces. It distributes the requests to the handler methods based on a type
// switch In case of no matching method a http.ErrMethodNotAllowed is returned.
type MethodHandler struct {
    handler http.Handler
}

func NewMethodHandler(h http.Handler) *MethodHandler {
    return &MethodHandler{
        handler: h,
    }
}

// ServeHTTP implements the http.Handler interface.
func (h *MethodHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        if hh, ok := h.handler.(GetHandler); ok {
            hh.ServeHTTPGet(w, r)
            return
        }
    case http.MethodPost:
        if hh, ok := h.handler.(PostHandler); ok {
            hh.ServeHTTPPost(w, r)
            return
        }
     case http.MethodPut:
        if hh, ok := h.handler.(PutHandler); ok {
            hh.ServeHTTPPut(w, r)
            return
        }
    case http.MethodDelete:
        if hh, ok := h.handler.(DeleteHandler); ok {
            hh.ServeHTTPDelete(w, r)
            return
        }
    // ...
    }
    // Fall back to default for no matching handler method or any
    // other HTTP method.
    h.handler.ServeHTTP(w, r)
}
```

## Example

### The new cache server package

* Re-implementing the cache handler using the method wrapper

```go
package cache

import (
    "log"
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

// Handler provides a simple in-memory cache server. Cache is done
// via a map of string to []byte. The sync.RWMutex is used to ensure that
// the cache is thread-safe.
type Handler struct {
    mu    sync.RWMutex
    cache map[string][]byte
}

// NewHandler creates the cache server. It's simply needed to create
// the map of string to []byte.
func NewHandler() *Handler {
    return &Handler{
        cache: make(map[string][]byte),
    }
}

// ServeHTTPGet handles GET requests. If the path can be found in the cache,
// its data is returned. Otherwise, the response is set to 404. Only
// read lock is needed.
func (h *Handler) ServeHTTPGet(w http.ResponseWriter, r *http.Request) {
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

// ...
```

### Using the cache server with the wrapper

```go
package main

import (
    "log"
    "net/http"

    "./pkg/cache"
    "./pkg/httpx"
)

// main runs the cache server.
func main() {
    h := cache..NewHandler()
    mh := httpx.NewMethodHandler(h) // Use the method handler as wrapper.
    err := http.ListenAndServe(":8080", mh)

    if err != nil {
        log.Fatal(err)
    }
}
```

## Links

* File [methods.go](https://github.com/tideland/go-httpx/blob/main/methods.go) of Tideland Go HTTPX
