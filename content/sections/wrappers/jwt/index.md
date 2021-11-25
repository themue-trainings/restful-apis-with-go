---
title: 'JSON Web Token'
summary: 'Use the JSON Web token for authentication and authorization.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 52
---

## JSON Web Token

* The JSON Web Token (JWT) is a compact, self-contained, signed JSON object that represents a security token
* It is a base64 encoded string that contains a header, a payload and a signature
* The header contains the type of token and the algorithm used to sign the token
* The payload contains the claims of the token
* The signature is used to verify the token
* It is a standard way to authenticate and authorize users
* The claims may contain an expiration date useful for session handling
* Several implementations are available, like [Tideland Go JSON Web Token](https://pkg.go.dev/tideland.dev/go/jwt)
* Login and retrieving of right credentials has to be done via redirection to the authorization URL

## HTTPX

### JWTHandler as a wrapper for other handlers

```go
package httpx

import (
    "encoding/json"
    "encoding/xml"
    "net/http"
    "strconv"
    "time"

    "tideland.dev/go/jwt"
)

// JWTHandlerConfig allows to control how the JWT handler works.
// All values are optional. In this case tokens are only decoded
// without using a cache, validated for the current time plus/minus
// a minute leeway, and there's no user defined gatekeeper function
// running afterwards.
type JWTHandlerConfig struct {
    Cache      *jwt.Cache
    Key        jwt.Key
    Leeway     time.Duration
    Gatekeeper func(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error
}

// JWTHandler checks for a valid token and then runs
// a gatekeeper function.
type JWTHandler struct {
    handler    http.Handler
    cache      *jwt.Cache
    key        jwt.Key
    leeway     time.Duration
    gatekeeper func(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error
}

// NewJWTHandler creates a handler checking for a valid JSON
// Web Token in each request.
func NewJWTHandler(handler http.Handler, config *JWTHandlerConfig) *JWTHandler {
    h := &JWTHandler{
        handler: handler,
        leeway:  time.Minute,
    }
    if config != nil {
        if config.Cache != nil {
            h.cache = config.Cache
        }
        if config.Key != nil {
            h.key = config.Key
        }
        if config.Leeway != 0 {
            h.leeway = config.Leeway
        }
        if config.Gatekeeper != nil {
            h.gatekeeper = config.Gatekeeper
        }
    }
    return h
}

// ServeHTTP implements the http.Handler interface. It checks for an existing
// and valid token before calling the wrapped handler.
func (h *JWTHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if h.isAuthorized(w, r) {
        h.handler.ServeHTTP(w, r)
    }
}

// isAuthorized checks the request for a valid token and if configured
// asks the gatekeepr if the request may pass.
func (h *JWTHandler) isAuthorized(w http.ResponseWriter, r *http.Request) bool {
    var token *jwt.JWT
    var err error
    switch {
    case h.cache != nil && h.key != nil:
        token, err = h.cache.RequestVerify(r, h.key)
    case h.cache != nil && h.key == nil:
        token, err = h.cache.RequestDecode(r)
    case h.cache == nil && h.key != nil:
        token, err = jwt.RequestVerify(r, h.key)
    default:
        token, err = jwt.RequestDecode(r)
    }
    // Now do the checks.
    if err != nil {
        h.deny(w, r, err.Error(), http.StatusUnauthorized)
        return false
    }
    if token == nil {
        h.deny(w, r, "no JSON Web Token", http.StatusUnauthorized)
        return false
    }
    if !token.IsValid(h.leeway) {
        h.deny(w, r, "the JSON Web Token claims 'nbf' and/or 'exp' are not valid", http.StatusForbidden)
        return false
    }
    if h.gatekeeper != nil {
        err := h.gatekeeper(w, r, token.Claims())
        if err != nil {
            h.deny(w, r, "access rejected by gatekeeper: "+err.Error(), http.StatusUnauthorized)
            return false
        }
    }
    // All fine.
    return true
}

// deny sends a negative feedback to the caller.
func (h *JWTHandler) deny(w http.ResponseWriter, r *http.Request, msg string, statusCode int) {
    feedback := map[string]string{
        "statusCode": strconv.Itoa(statusCode),
        "message":    msg,
    }
    accept := r.Header.Get(HeaderAccept)
    w.WriteHeader(statusCode)
    err := WriteBody(w, accept, feedback)    
    if err != nil {
        logger.Errorf("JWT handler: %v", err)
    }
}
```

### Example

* `NestedMux` wraps several business logic handlers
* `LoggingHandler` wraps the `NestedMux` and logs the requests
* `JWTHandler` wraps the `LoggingHandler` and checks for a valid token
* `ServeMux` wraps the `LoggingHandler` and serves all requests
* Gatekeeper here only looks for existing claim `access` with the value `allowed`

```go
package main

import (
    "log"
    "net/http"
    "os"

    "./pkg/httpx"
    "./pkg/user"
)

func main() { 
    mux := http.NewServeMux()
    apimux := httpx.NewNestedMux("/api/v1")
    apilogger := httpx.NewLoggingHandler(log.New(os.Stdout, "api: ", log.LstdFlags), apimux)
    gatekeeper := func(w http.ResponseWriter, r *http.Request, claims jwt.Claims) error {
        access, ok := claims.GetString("access")
        if !ok || access != "allowed" {
            return errors.New("access is not allowed")
        }
        return nil
    }
    apijwt := httpx.NewJWTHandler(apilogger, &httpx.JWTHandlerConfig{
        Key: []byte("secret"),
        Gatekeeper: gatekeeper,
    })

    apimux.Handle("users", httpx.MethodWrapper(user.NewUsersHandler()))
    apimux.Handle("users/addresses", httpx.MethodWrapper(user.NewUsersAddressesHandler()))
    apimux.Handle("users/contracts", httpx.MethodWrapper(user.NewUsersContractsHandler()))

    // Register further nested API handlers ...

    mux.Handle("/api/v1/", apijwt)

    // Register further multiplexed handlers ...

    http.ListenAndServe(":8080", mux)
}
```
