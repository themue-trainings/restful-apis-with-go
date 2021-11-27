---
title: 'Producing Web Responses'
summary: 'Answer web requests with the http.ResponseWriter.'
date: 2021-11-26T12:00:00+1:00
draft: false
weight: 23
---

## Responses

## http.ResponseWriter

* The type `http.ResponseWriter` is an interface type that represents the response being sent back to the client
* It defines only three methods
    * `Header()`: returns the header map and allows you to set the response header
    * `WriteHeader(int)`: sets the response status code
    * `Write([]byte) (int, error)`: writes the response body
* `Header()` and `WriteHeader()` have to be called before writing the body
* The method `Write()` ensures that the response also implements the `io.Writer` interface
* So all functions and methods working with `io.Writer` can also be used writing to the `http.ResponseWriter`

## Examples

### Hello, World!

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/plain")
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Hello, World!"))
}
```

### Accept

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    accept := r.Header.Get("Accept")

    w.Header().Set("Content-Type", "text/plain")
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "Your Accept header is: %s", accept)
}
```

## Links

* Type [http.ResponseWriter](https://pkg.go.dev/net/http#ResponseWriter)

