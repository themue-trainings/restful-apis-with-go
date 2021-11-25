---
title: 'Usage of nested Handlers'
summary: 'An example on how to use the multiplexer of HTTPX for nesting.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 43
---

## Idea

* Register the RESTful API for users

## Example

### Use the nested multiplexer

* `NestedMux` wraps several business logic handlers
* `ServeMux` wraps the `NestedMux` and serves all requests

```go
package main

import (
    "net/http"

    "./pkg/httpx"
    "./pkg/users"
)

func main() { 
    mux := http.NewServeMux()
    apimux := httpx.NewNestedMux("/api/v1")

    apimux.Handle("users", httpx.MethodWrapper(users.NewHandler()))
    apimux.Handle("users/addresses", httpx.MethodWrapper(users.NewAddressesHandler()))
    apimux.Handle("users/contracts", httpx.MethodWrapper(users.NewContractsHandler()))

    // Register further nested API handlers ...

    mux.Handle("/api/v1/", apimux)

    // Register further multiplexed handlers ...

    http.ListenAndServe(":8080", mux)
}
```

