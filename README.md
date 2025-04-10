# Go Guidelines

Go Guidelines are a collection of practices, recommendations, and techniques for the Go language that I developed based
on my own experience. A significant portion of the material below was also _borrowed_ from other sources. While some of
these practices can be applied to other programming languages, my focus is specifically on Go.

<!-- TOC -->
* [Go Guidelines](#go-guidelines)
  * [General](#general)
    * [Avoid functions with internal state](#avoid-functions-with-internal-state)
  * [Patterns](#patterns)
    * [Avoid Functional Options](#avoid-functional-options)
    * [Sum types](#sum-types)
  * [Style](#style)
    * [Last word single letter receiver name](#last-word-single-letter-receiver-name)
  * [Testing](#testing)
    * [Test with `GOMAXPROCS` set to 1](#test-with-gomaxprocs-set-to-1)
    * [Use containers instead of mocks](#use-containers-instead-of-mocks)
<!-- TOC -->

## General

### Avoid functions with internal state

`NewUser` function in the example below has internal state that you won't be able to cover in the unit tests.

```go
package user

import (
	"time"
	"github.com/google/uuid"
)

type User struct {
	Name      string
	ID        uuid.UUID
	CreatedAt time.Time
}

type Service struct{}

func (s *Service) NewUser(name string) User {
	return User{
		Name:      name,
		ID:        uuid.New(),
		CreatedAt: time.Now(),
	}
}
```

Ideally in the unit test you want to create an expected instance of `User` and compare it with the result of the
function result.

The solution is to inject time and ID providers into the service and use them for a full control of the state.
The [gofrs/uuid](https://github.com/gofrs/uuid) package offers
the [Gen](https://pkg.go.dev/github.com/gofrs/uuid/v5#Gen) struct, which allows you to create a deterministic UUID
generator. For generating `time.Time` values, you can use
the [jonboulle/clockwork](https://github.com/jonboulle/clockwork) package.

## Patterns

### Avoid Functional Options

Default to using a simple Config struct as one of the parameters in your constructor.

```go
package server

import (
	"time"
)

type Config struct {
	Addr        string
	ConnTimeout time.Duration
}

type Server struct{}

func NewServer(cfg Config) (Server, error) {
	// skipped...
}
```

Using a configuration struct offers better discoverability of an object's options during creation. This approach
eliminates the need to write `With***` methods for each additional option. While the functional options pattern is
elegant, it shouldn't be your default choice, particularly when working on projects that aren't libraries.

### Sum types

Go doesn't have native support for sum types (discriminated unions). However, the pattern itself is quite useful and
still can be achieved with a few caveats.

```go
package user

type User interface {
	isUser()
}

type Customer struct{}

func (c Customer) isUser() {}

type Vendor struct{}

func (v Vendor) isUser() {}
```

This example shows a `User` interface with one sealed method and two structs implementing it. This approach ensures that
only structs within the `user` package can implement the method. When combined with
a [linter](https://github.com/alecthomas/go-check-sumtype) that checks for exhaustive switch case blocks, you
effectively get sum types in Go.

### Avoid oneof struct pattern

Use [sum types](#sum-types) instead. Usually oneof pattern is implemented using multiple optional fields inside a single
struct, that acts as a value holder.

```go
package user

type Value struct {
	String  *string
	Integer *int
}
```

While this pattern may be frustrating due to the extra attention needed when accessing fields, there's another concern:
any consumer of the struct can potentially assign values to multiple fields simultaneously, which would violate the 
pattern's intended behavior. Needless to say, that in some cases it will require more boilerplate code to do the proper
validation of the struct and `switch-casing`.

## Style

### Last word single letter receiver name

Using the last word's initial letter as a receiver name creates consistency across different interface implementations.
This naming convention is straightforward to follow. Moreover, when you generally avoid single-letter variables in your
code, you can instantly recognize that a single letter like 'x' represents a receiver variable.

```go
package service

type PostgresStore struct{}

func (s *PostgresStore) Save() {}

type UserService struct{}

func (s *UserService) CreateUser() {}
```

## Testing

### Test with `GOMAXPROCS` set to 1

When testing code that involves concurrency, you might encounter a situation where tests consistently pass on your local
machine but occasionally fail during CI/CD. One common reason for this is that CI/CD environments often allocate less
than one CPU core to execute tests. As a result, certain operations you expect to complete in a specific order might
actually execute later than anticipated, leading to unexpected behavior and test failures.

As a workaround, you can set `GOMAXPROCS` to `1` to help identify which assertions are failing in the test. Once you've
pinpointed the issue, consider using `assert.Eventually` from
the [testify/assert](https://github.com/stretchr/testify/assert)
package. This function allows you to check conditions over an extended period rather than expecting immediate results,
which is particularly useful for asynchronous operations.

### Use containers instead of mocks

When testing an implementation of an interface that interacts directly with external services (such as a database,
cache, or message broker), it's best to use the corresponding container image instead of a mock, stub, or fake —
whenever possible. Here's why:

* Mocks can behave differently from real service instances, leading to inaccurate or misleading test results.
* Mocks don’t perform real network communication, which might hide potential networking issues you'd encounter in
  production.
* To mimic the behavior of the actual service (e.g., SQL engine), you may end up reimplementing complex
  functionality—making your tests harder to maintain and potentially flawed.

In short, mocks typically fall short of accurately reflecting real-world conditions and move you further away from a
production-like environment.

Tools like [testcontainers-go](https://github.com/testcontainers/testcontainers-go)
and [dockertest](https://github.com/ory/dockertest) can help set up and manage lightweight, disposable containers during
testing.
