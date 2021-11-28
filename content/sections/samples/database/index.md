---
title: 'Connect a Database'
summary: 'Create a handler providing a RESTful API for a customer database.'
date: 2021-11-26T12:00:00+1:00
draft: false
weight: 81
---

## Idea

* We've got a service type responsible for managing a customer database
* In the example It resides in the same package and is only for encapsulating the database

## Implementation

```go
package customer

import (
    "net/http"

    "./pkg/httpx"
)

const (
    // Prefix for API calls.
    apiPrefix = "/api/v1"
    
    // Name of the customer resource.
    ressourceName = "customer"
)

type Handler struct {
    service *Service
}

// NewHandler returns the wrapped web handler for the customer handler.
func NewHandler(service *Service) http.Handler {
    return httpx.NewMethodHandler(&Handler{
        service: service,
    })
}

// ServeHTTPPost creates the customer.
func (h *Handler) ServeHTTPPost(w http.ResponseWriter, r *http.Request) {
    var c Customer
    err := httpx.ReadBody(r, &c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    if !httpx.PathToResources(apiPrefix).IsPath(ressourceName) {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    err = h.service.Create(c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Error(w, "", http.StatusCreated)
}

// ServeHTTPGet reads the customer.
func (h *Handler) ServeHTTPGet(w http.ResponseWriter, r *http.Request) {
    ress := httpx.PathToResources(r, apiPrefix)
    if !ress.IsPath(ressourceName) {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    if ress[0].ID == "" {
        // Handling of queries is still missing in httpx. Needs a kind of
        // query parsing like
        //
        // args := httpx.QueryToArguments(r)
        // cs, err := h.service.Find(args)   
        http.Error(w, http.StatusText(http.StatusNotImplemented), http.StatusNotImplemented)
        return
    }
    c, err := h.service.Read(ress[0].ID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    err = httpx.WriteBody(w, c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// ServeHTTPPut updates the customer.
func (h *Handler) ServeHTTPPut(w http.ResponseWriter, r *http.Request) {
    var c Customer
    err := httpx.ReadBody(r, &c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    ress := httpx.PathToResources(r, apiPrefix)
    if !ress.IsPath(ressourceName) {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    err = h.service.Update(ress.PathID(ressourceName), c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Error(w, "", http.StatusNoContent)
}

// ServeHTTPDelete deletes the customer.
func (h *Handler) ServeHTTPDelete(w http.ResponseWriter, r *http.Request) {
    ress := httpx.PathToResources(r, apiPrefix)
    if !ress.IsPath(ressourceName) {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    err := h.service.Delete(ress.PathID(ressourceName))
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Error(w, "", http.StatusNoContent)
}
```
