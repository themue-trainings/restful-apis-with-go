---
title: 'CRUD and HTTP Methods'
summary: 'Description of how CRUD methods are mapped to HTTP methods in database oriented applications.'
date: 2021-11-21T12:00:00+1:00
draft: false
weight: 32
---

## CRUD

* CRUD methods are used to create, read, update and delete resources
* HTTP methods are mapped to CRUD methods
    * POST: Create a new resource
    * GET: Read a resource
    * PUT: Update a resource
    * DELETE: Delete a resource
* RESTful APIs are idempotent, meaning that they can be called multiple times without changing the resource
* This is especially important in designing the data structures for creating and updating resources

## Other HTTP methods

* Other HTTP methods are used to perform more operations on resources
* They are handled by the HTTP methods HEAD, PATCH, OPTIONS, TRACE and CONNECT of RESTful APIs:
    * HEAD: Used to check if the resource exists and to get the resource's metadata
    * PATCH: Can be used to update a resource partially
    * OPTIONS: Used to check if the resource exists and to get the resource's metadata
    * TRACE: Used for diagnostics and debugging
    * CONNECT: Used to connect to a remote resource

## URI format and components

* URI is the Uniform Resource Identifier
* It is composed of the following parts:
    * Scheme: The protocol used to access the resource
    * Host: The domain name or IP address of the resource
    * Path: The path of the resource
    * Query: The query string of the resource
* The path is used to identify the resource
* Arguments inside the query string are used to narrow or order requests of resource lists
* This data is provided by `url.URL` inside the `http.Request` object
