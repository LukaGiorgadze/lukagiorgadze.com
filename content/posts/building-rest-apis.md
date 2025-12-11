---
title: "Building REST APIs with Go"
date: 2024-12-05
draft: false
tags: ["go", "api", "rest", "web"]
categories: ["Programming"]
summary: "Learn how to build production-ready REST APIs using Go's standard library and popular frameworks."
---

Building REST APIs in Go is straightforward thanks to its powerful standard library. In this post, we'll explore how to create a simple but production-ready API.

## Using the Standard Library

Go's `net/http` package provides everything you need to build a web server:

```go
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func main() {
    http.HandleFunc("/users", getUsers)
    http.ListenAndServe(":8080", nil)
}
```

## Adding a Router

For more complex routing, consider using a router like `chi` or `gorilla/mux`:

```go
import "github.com/go-chi/chi/v5"

func main() {
    r := chi.NewRouter()

    r.Get("/users", getUsers)
    r.Get("/users/{id}", getUser)
    r.Post("/users", createUser)

    http.ListenAndServe(":8080", r)
}
```

## Best Practices

1. **Use middleware** for logging, authentication, and error handling
2. **Validate input** before processing requests
3. **Return proper status codes** for different scenarios
4. **Document your API** using OpenAPI/Swagger

## Conclusion

Go makes it easy to build fast, reliable REST APIs. Start with the standard library and add dependencies only when needed.
