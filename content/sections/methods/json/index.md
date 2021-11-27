---
title: 'JSON as Lingua Franca'
summary: 'Introduction how JSON is used and Go handles it.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 34
---

## The data format JSON

* RESTful APIs can be used to send data in different formats like JSON, XML, CSV, etc.
* JSON is the most popular format
* It is a text-based format representing data in a form of a sequence of key-value pairs
* RESTful API requests and responses containing JSON data use the header `Content-Type: application/json`

```json
{
    "firstName": "John",
    "lastName": "Doe",
    "age": 30,
    "cars": [ "Ford", "BMW", "Fiat" ]
}
```

## Go and JSON

* Go supports JSON encoding and decoding using the `encoding/json` package
* Data structures to be encoded and decoded must contain exported fields
* Names and behavior in case of empty values are defined by the `json` struct tags

```go
type Person struct {
    FirstName string   `json:"firstName"`
    LastName  string   `json:"lastName"`
    Age       int      `json:"age"`
    Cars      []string `json:"cars,omitempty"` // omitempty is used to omit the field if it is empty.

    privateData string // privateData field is not exported.
}
```

### Marshalling

* Marshalling just needs the data structure to be encoded
* It returns the JSON representation of the data structure and a potential error
* The JSON representation of the data structure is returned as a byte slice
* HTTP response needs the according content type and the byte slice

```go
person := Person{
    FirstName: "John",
    LastName:  "Doe",
    Age:       30,
    Cars:      []string{"Ford", "BMW", "Fiat"},
}

// Marshal the person struct to JSON.
jsonPerson, err := json.Marshal(person)
if err != nil {
    log.Fatal(err)
}

// Write the JSON.
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
w.Write(jsonPerson)
```

### Unmarshalling

* HTTP request body contains the JSON representation of the data structure
* The header should be tested to ensure that the content type is `application/json`
* Unmarshal parses the JSON-encoded data and stores the result in the pointed value
* If the value is nil or not a pointer or the JSON is not valid, `Unmarshal` returns an `InvalidUnmarshalError`

```go
// Check the header.
if r.Header.Get("Content-Type") != "application/json" {
    http.Error(w, "Invalid request", http.StatusUnsupportedMediaType)
    return
}

// Unmarshal the JSON to the person struct.
var person Person
err := json.Unmarshal(r.Body, &person)
if err != nil {
    http.Error(w, "Invalid request", http.StatusBadRequest)
    return
}

// Continue with the request.
...
``` 

## Links

* Package [encoding/json](https://golang.org/pkg/encoding/json/)