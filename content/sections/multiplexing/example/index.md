---
title: 'Usage of nested Handlers'
summary: 'An example on how to use the multiplexer of HTTPX for nesting.'
date: 2021-11-28T12:00:00+1:00
draft: false
weight: 43
---

## Idea

* Register a RESTful API for customers

## Example

### Use the nested multiplexer

* `NestedMux` wraps several business logic handlers
* `ServeMux` wraps the `NestedMux` and serves all requests

```go
package main

import (
    "net/http"

    "./pkg/httpx"
    "./pkg/customers"
)

const (
    // Prefix for API calls.
    apiPrefix = "/api/v1"
)

func main() { 
    mux := http.NewServeMux()
    apimux := httpx.NewNestedMux(apiPrefix)

    apimux.Handle("customers", httpx.MethodWrapper(customers.NewHandler()))
    apimux.Handle("customers/addresses", httpx.MethodWrapper(customers.NewAddressesHandler()))
    apimux.Handle("customers/contracts", httpx.MethodWrapper(customers.NewContractsHandler()))

    // Register further nested API handlers, then
    // let the multiplexer serve all API requests.

    mux.Handle(apiPrefix, apimux)

    // Register further multiplexed handlers, e.g. for static files.

    http.ListenAndServe(":8080", mux)
}
```

