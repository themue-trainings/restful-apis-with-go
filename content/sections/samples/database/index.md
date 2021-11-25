---
title: 'Connect a Database'
summary: 'Create a handler providing a RESTful API for a customer database.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 81
---

## Idea

* We've got a service type responsible for managing a customer database
* In the example It resides in the same package and is only for encapsulating the database
* Also checking if resources is `customer` is missing and should be added

## Implementation

```go
package customer

import (
    "net/http"

    "./pkg/httpx"
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
    if !httpx.PathToResources("/api/v1").IsPath("customer") {
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
    rs := httpx.PathToResources(r, "/api/v1")
    if !rs.IsPath("customer") {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    if rs[0].ID == "" {
        // Handling of queries is still missing in httpx. Needs a kind of
        // query parsing like
        //
        // args := httpx.QueryToArguments(r)
        // cs, err := h.service.Find(args)   
        http.Error(w, http.StatusText(http.StatusNotImplemented), http.StatusNotImplemented)
        return
    }
    c, err := h.service.Read(rs[0].ID)
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
    rs := httpx.PathToResources(r, "/api/v1")
    if !rs.IsPath("customer") {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    err = h.service.Update(rs[0].ID, c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Error(w, "", http.StatusNoContent)
}

// ServeHTTPDelete deletes the customer.
func (h *Handler) ServeHTTPDelete(w http.ResponseWriter, r *http.Request) {
    rs := httpx.PathToResources(r, "/api/v1")
    if !rs.IsPath("customer") {
        http.Error(w, "missing or bad resource", http.StatusBadRequest)
        return
    }
    err := h.service.Delete(rs[0].ID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Error(w, "", http.StatusNoContent)
}
```
