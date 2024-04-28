---
layout: post
title: Iterators and Generators in Go
date: 2024-04-21
tags: [go, appdev]
---

## Introduction

Something I've found missing in Go is built-in generator functions and an
iterator interface. Python & JavaScript both have them and I find myself using
them regularly. Having syntactic sugar to express a generator & iterators is
great for consistency sake, and also makes them more accessible for
implementers.

## What are generator functions?

A generator function produces values on the fly or "lazily", instead of
generating them all at once and storing them in memory.

In most languages/implementations, it typically utilises the `yield` keyword to
temporarily suspend execution and "yield" a value to the caller, allowing the
function to be resumed later, efficiently maintaining its state.

To a consumer iterating over the function, it behaves similar to a function
that returns an array, making it easy to consume.

Generators significantly reduce memory overhead and improve performance where
memory constraints are a concern, or when dealing with computationally expensive
operations. Iterators created with generator functions facilitate clean and
concise code, which improves readability and maintainability.

## Go 1.22 rangefunc experiment

Luckily Go 1.22 shipped with [a preliminary implementation for function
iterators](https://tip.golang.org/wiki/RangefuncExperiment). It can be enabled
by setting `GOEXPERIMENT=rangefunc` when building your Go program.

Using VS Code? Add this to your `settings.json` to fix the errors displayed:

```json
"go.toolsEnvVars": {
    "GOEXPERIMENT": "rangefunc"
}
```

I've been using it the past month and found it pleasant. Below I'll quickly
explain the interface and also show some examples.

### the simplest generator

A generator function which yields integers would have this interface:

```go
func (yield func(int) bool)
```

Where the function will call the exposed `yield` function to "yield" results,
and return to complete.

Below is an example of the above function prototype with a rate limited
fibonacci generator

```go
package main

import (
    "fmt"
    "time"
)

func Fibonacci(yield func(int) bool) {
    a, b := 0, 1
    for {
        a, b = b, a+b
        if !yield(a) {
            return
        }
    }
}

func main() {
    for num := range Fibonacci {
        fmt.Println(num)
        time.Sleep(200 * time.Millisecond)
    }
}
```

### checking the return value of `yield()`

Why must we check the return value of `yield()` ? This is how the generator
knows it should stop yielding values, and can clean up anything before ending
execution by returning. If the function ignores the return value of `yield()`
and continues it will `panic`.

### yielding an error

We can also yield up to two values (but no more than 2!)

e.g:
```go
func (yield func(int, error) bool)
```

This makes it great for returning errors in our second argument, then the
generator can clean up any resources and `return` to complete execution.

### what about parameters?

The generator function prototype above does not allow you provide any
parameters, which in most situations would be pretty useless. Fortunately the
new `iter` package returns `iter.Seq` and `iter.Seq2` types, which you can use
in a parent function to return your iterator.

A simple example which yields the same string parameter (with value `ahh`) ten
times looks like this:

```go
func Generate(value string) iter.Seq[string] {
    return func(yield func(string) bool) {
        for i := 1; i <= 10; i++ {
            if !yield(value) {
                return
            }
        }
    }
}

func main() {
    for x := range Generate("ahh") {
        fmt.Println(x)
    }
}
```


### an example with http

Here's an example hitting a paginated HTTP endpoint and yielding the data.
Errors also yield back to the consumer stop execution.

```go
package main

import (
    "encoding/json"
    "fmt"
    "iter"
    "net/http"
)

type Pokemon struct {
    Name string `json:"name"`
    URL  string `json:"url"`
}
type APIResponse struct {
    Count    int       `json:"count"`
    Next     *string   `json:"next"`
    Previous *string   `json:"previous"`
    Results  []Pokemon `json:"results"`
}

func Generate(limit int) iter.Seq2[Pokemon, error] {
    return func(yield func(Pokemon, error) bool) {
        var data APIResponse
        url := fmt.Sprintf("https://pokeapi.co/api/v2/pokemon?limit=%d&offset=0", limit)
        for {
            response, err := http.Get(url)
            if err != nil {
                yield(Pokemon{}, err)
                return
            }
            defer response.Body.Close()

            err = json.NewDecoder(response.Body).Decode(&data)
            if err != nil {
                yield(Pokemon{}, err)
                return
            }
            for _, pokemon := range data.Results {
                if !yield(pokemon, nil) {
                    return
                }
            }

            if data.Next == nil {
                break
            }
            url = *data.Next
        }
    }
}

func main() {
    for pokemon, err := range Generate(10) {
        if err != nil {
            panic(err)
        }
        // stop anytime
        fmt.Println("Name:", pokemon.Name, "URL:", pokemon.URL)
    }
}
```

## See also

There's plenty more features from the `rangefunc` experiment mentioned in the
below links. Check them out!

* Go specs/proposals:
    * <https://tip.golang.org/wiki/RangefuncExperiment>
    * <https://github.com/golang/go/issues/61897>
    * <https://github.com/golang/go/issues/61897>
* Blogs:
    * <https://medium.com/eureka-engineering/a-look-at-iterators-in-go-f8e86062937c>
    * <https://bitfieldconsulting.com/golang/iterators>
    * <https://blog.perfects.engineering/go_range_over_funcs>
    * <https://makubob.medium.com/exploring-the-upcoming-go-generators-344b2fb98ff9>
