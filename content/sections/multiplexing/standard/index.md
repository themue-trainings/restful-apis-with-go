---
title: 'The Standard Multiplexer'
summary: 'How requests are distibuted to different handlers with the Go standard library.'
date: 2021-11-28T12:00:00+1:00
draft: false
weight: 41
---

## The http.ServeMux multiplexer

* `http.ServeMux` is an HTTP request multiplexer
* It matches the URL of each incoming request against a list of registered patterns
* It calls the handler for the pattern that most closely matches the URL
* For example, the pattern `/static/` matches all paths that begin with `/static/`
* The pattern `/` matches all paths
* It implements the `http.Handler` interface, so that it can be used as a handler

## Examples

### Simple static web server

```go
package main

import (
    "fmt"
    "net/http"
)

func index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to the home page!")
}

func main() {
    mux := http.NewServeMux()

    mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("/var/www"))))
    mux.HandleFunc("/", index)

    http.ListenAndServe(":8080", mux)
}
```

### APIs

```go
package main

import (
    "net/http"

    "./pkg/cache"
    "./pkg/httpx"
    "./pkg/jsoncache"
    "./pkg/users"
)

func main() { 
    mux := http.NewServeMux()

    mux.Handle("/api/v1/cache/", httpx.MethodWrapper(cache.NewHandler()))
    mux.Handle("/api/v1/json-cache/", httpx.MethodWrapper(jsoncache.NewHandler()))
    mux.Handle("/api/v1/users/", httpx.MethodWrapper(users.NewHandler()))

    // ...

    http.ListenAndServe(":8080", mux)
}
``` 

## Links

* Type [http.ServeMux](https://pkg.go.dev/net/http#ServeMux)
