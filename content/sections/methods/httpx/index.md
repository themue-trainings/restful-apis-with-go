---
title: 'Body handling in httpx'
summary: 'Extending the httpx package with to handle JSON - and other - bodies.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 35
---

## Receiving request bodies

* Add a function taking care for the content type of the request body
* Shall be flexible enough to handle typical ones

```go
package httpx

import (
    "encoding/base64"
    "encoding/json"
    "encoding/xml"
    "io/ioutil"
    "net/http"
)

const (
    HeaderAccept      = "Accept"
    HeaderContentType = "Content-Type"

    ContentTypeJSON = "application/json"
    ContentTypeText = "text/plain"
    ContentTypeXML  = "application/xml"
)

// ReadBody reads and unmarshals the body of the request into the given interface. It
// analyzes the content type and uses the appropriate unmarshaler. Here it handles
// plain text, JSON, and XML. All other content types are returned directly as byte slice.
func ReadBody(r *http.Request, v interface{}) error {
    // Read content type and body.
    contentType := r.Header.Get(HeaderContentType)
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        return err
    }
    if err = r.Body.Close(); err != nil {
        return err
    }

    // Unmarshal based on content type.
    switch contentType {
    case ContentTypeJSON:
        return json.Unmarshal(body, v)
    case ContentTypeText:
        pv, ok := v.(*string) // Assume v is a string pointer.
        if !ok {
            return fmt.Errorf("ReadBody: v is not a string pointer")
        }
        *pv = string(body)
    case ContentTypeXML:
        return xml.Unmarshal(body, v)
    default:
        pbs, ok := v.(*[]byte) // Assume v is a byte slice pointer.
        if !ok {
            return fmt.Errorf("ReadBody: v is not a byte slice pointer")
        }
        *pbs = body
    }
    return nil
}
```

## Sending response bodies

* Marshal a value based on the given content type
* Send it to the client via response writer

```go
// WriteBody writes the given value to the response writer. It analyzes the content
// type and uses the the appropriate encoding. Here it handles plain text, JSON, and
// XML. All other content types are written directly as byte slice.
func WriteBody(w http.ResponseWriter, contentType string, v interface{}) error {
    // Marshal based on content type.
    switch contentType {
    case ContentTypeJSON:
        body, err := json.Marshal(v)
        if err != nil {
            return err
        }
        w.Header().Set(HeaderContentType, ContentTypeJSON)
        w.Write(body)
    case ContentTypeText:
        s, ok := v.(string)
        if !ok {
            return fmt.Errorf("WriteBody: v is not a string")
        }
        w.Header().Set(HeaderContentType, ContentTypeText)
        w.Write([]byte(s))
    case ContentTypeXML:
        body, err := xml.Marshal(v)
        if err != nil {
            return err
        }
        w.Header().Set(HeaderContentType, ContentTypeXML)
        w.Write(body)
    default:
        bs, ok := v.([]byte)
        if !ok {
            return fmt.Errorf("WriteBody: v is not a byte slice")
        }
        w.Header().Set(HeaderContentType, contentType)
        w.Write(bs)
    }
    return nil
}
```

## Links

* File [body.go](https://github.com/tideland/go-httpx/blob/main/body.go) of Tideland Go HTTPX
