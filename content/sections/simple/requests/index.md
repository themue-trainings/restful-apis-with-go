---
title: 'Analyzing Web Requests'
summary: 'Using http.Request and the related types for the analysis of web requests.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 22
---

## Requests

### http.Request

* The type `http.Request` represents an HTTP request received by a server
* It also can be sent to a server with the `http.Client`
* It contains information about the client request, such as the request method, URL, headers, and so on
* Important fields:
    * `Method`: the HTTP method of the request (GET, POST, PUT, etc.)
    * `URL`: the URL of the request
    * `Header`: the HTTP headers of the request
    * `Body`: the body of the request
    * `Host`: the host of the request
    * `RemoteAddr`: the remote address of the request
* Some fields are simple data types, like `string` or `int`
* Others are complex data types, like `url.URL`, `http.Header`, or `multipart.Form`

### url.URL

* The type `url.URL` represents a parsed URL
* It contains information about the URL, such as the scheme, host, path, and so on
* Functions and methods help to parse and create URLs

### http.Header

* The type `http.Header` represents an HTTP message header
* It is a map of header field names to values
* Output to an `io.Writer` as a series of HTTP header lines in the format "Header-Name: value\r\n"

### multipart.Form

* The type `multipart.Form` represents an HTTP request body which uses the `multipart/form-data` MIME format
* It is a map of field names to list of values
* It also contains a `File` map, which is a map of field names to list of `multipart.FileHeader`
* It can be used to parse the request body of a POST request

## Examples

### Method switch

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        h.handleGet(w, r)
    case http.MethodPost:
        h.handlePost(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}
```

### Multipart form

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Parse the request body.
    err := r.ParseMultipartForm(10 << 20) // 10 MB
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Get the form value.
    value := r.FormValue("field-name")

    // Get the file header.
    fileHeader := r.MultipartForm.File["field-name"][0]

    // Get the file.
    file, err := fileHeader.Open()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer file.Close()

    // Do something with the file.

    ...
}
```
