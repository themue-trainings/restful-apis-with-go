---
title: 'Keep the door'
summary: 'Allow entrance to functionality based on HTTP method and RESTful API path.'
date: 2021-11-28T12:00:00+1:00
draft: false
weight: 82
---

## Idea

* Provide a gatekeeper function for the `JWTHandler`
* It shall check for a claim named `rights`
* This claim contains a map of allowed paths to slices of allowed HTTP methods
* An authorization handler has to create it based on the actual rights of the user

## Implementation

```go
package gatekeeper

import (
    "fmt"
    "net/http"

    "tideland.dev/go/jwt"
)

const (
    // Prefix for API calls.
    apiPrefix = "/api/v1"

    // Claim name for the rights.
    rightsClaim = "rights"
)

// NewGatekeeper returns a gatekeeper using the given authorization backend.
func NewGatekeeper(auth *Authorization) Gatekeeper func(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error {
    return func(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error {
        rights, ok := claims.GetString(rightsClaim)
        if !ok {
            return fmt.Errorf("no rights claim found")
        }
        ress := httpx.PathToResources(r, apiPrefix)
        if ress == nil {
            return fmt.Errorf("resource and ID are missing")
        }
		if !auth.IsAllowed(r.Method, ress, rights) {
            return fmt.Errorf("method %q and ressources %q are not allowed", r.Method, ress.Path())
       	}
        return nil
    }
}
```
