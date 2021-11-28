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

// Rights contains the allowed paths and their allowed HTTP methods.
type Rights struct {
    Paths map[string][]string `json:"paths"`
}

// Gatekeeper retrieves the claim "rights" and tests if the path and method are allowed.
func Gatekeeper(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error {
    var rights Rights
    ok, err := claims.GetMarshalled(rightsClaim, &rights)
    if err != nil {
        return fmt.Errorf("cannot unmarshal rights: %w", err)
    }
    if !ok {
        return fmt.Errorf("no rights claim found")
    }
    ress := httpx.PathToResources(r, apiPrefix)
    if ress == nil {
        return fmt.Errorf("resource and ID are missing")
    }
    for _, method := range rights.Paths[ress.Path()] {
        if method == r.Method {
            // Method in path is found, so access allowd.
            return nil
        }
    }
    return fmt.Errorf("path %q and method %q are not allowed", ress.Path(), r.Method)
}
```