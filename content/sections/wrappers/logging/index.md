---
title: 'Wrapper to log Requests'
summary: 'Show the pattern of wrappers with an example for logging of web requests.'
date: 2021-11-28T12:00:00+1:00
draft: false
weight: 51
---

## Continue the idea of wrapping

* The interface `http.Handler` allows to wrap any HTTP handler easily
* This way it's ideal to separate functional and non-functional code
* While our own handlers take care for the business logic wrappers like `httpx.MethodHandler` and `httpx.NestedMux` take care for the HTTP logic
* This can be continued with other aspects of the application, like the logging

## HTTPX

```go
package httpx

import (
    "net/http"
)

// Logger defines an interface for many different loggers. So e.g.
// the log.Logger from the standard library or the logrus.Logger can
// be used.
type Logger interface {
    Printf(format string, v ...interface{})
}

// LoggingHandler wraps handlers and logs the requests to them.
type LoggingHandler struct {
    logger  Logger
    handler http.Handler
}

// NewLoggingHandler creates a new logging handler with the given logger and handler.
func NewLoggingHandler(logger Logger, handler http.Handler) *LoggingHandler {
    return &LoggingHandler{
        logger:  logger,
        handler: handler,
    }
}

// ServeHTTP logs the request and calls the wrapped handler.
func (h *LoggingHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.logger.Printf("%s %s", r.Method, r.URL.Path)
    h.handler.ServeHTTP(w, r)
}
```

## Example

* `NestedMux` wraps several business logic handlers
* `LoggingHandler` wraps the `NestedMux` and logs the requests
* `ServeMux` wraps the `LoggingHandler` and serves all requests

```go
package main

import (
    "log"
    "net/http"
    "os"

    "./pkg/httpx"
    "./pkg/user"
)

const (
    // Prefix for API calls.
    apiPrefix = "/api/v1"
)

func main() { 
    mux := http.NewServeMux()
    apimux := httpx.NewNestedMux(apiPrefix)
    apilogger := httpx.NewLoggingHandler(log.New(os.Stdout, "api: ", log.LstdFlags), apimux)

    apimux.Handle("users", httpx.MethodWrapper(user.NewUsersHandler()))
    apimux.Handle("users/addresses", httpx.MethodWrapper(user.NewUsersAddressesHandler()))
    apimux.Handle("users/contracts", httpx.MethodWrapper(user.NewUsersContractsHandler()))

    // Register further nested API handlers, then
    // let the multiplexer serve all API requests.

    mux.Handle(apiPrefix, apilogger)

    // Register further multiplexed handlers, e.g. for static files.

    http.ListenAndServe(":8080", mux)
}
```

## Links

* File [logging.go](https://github.com/tideland/go-httpx/blob/main/logging.go) of Tideland Go HTTPX
