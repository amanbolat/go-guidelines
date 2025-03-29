# Go Guidelines

Go Guidelines are a collection of practices, recommendations, and techniques for the Go language that I developed based on my own experience. A significant portion of the material below was also _borrowed_ from other sources. While some of these practices can be applied to other programming languages, my focus is specifically on Go.

<!-- TOC -->
* [Go Guidelines](#go-guidelines)
  * [General](#general)
    * [Avoid functions with internal state](#avoid-functions-with-internal-state)
  * [Type system](#type-system)
    * [Sum types](#sum-types)
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
)

type User struct {
	Name      string
	ID        uuid.UUID
	CreatedAt time.Time
}

type Service struct {}

func (s *Service) NewUser(name string) User {
	return User{
		Name:      name,
		ID:        uuid.New(),
		CreatedAt: time.Now(),
	}
}
```

Ideally in the unit test you want to create an expected instance of `User` and compare it with the result of the function result.

The solution is to inject time and ID providers into the service and use them for a full control of the state. The [gofrs/uuid](https://github.com/gofrs/uuid) package offers the [`Gen`](https://pkg.go.dev/github.com/gofrs/uuid/v5#Gen) struct, which allows you to create a deterministic UUID generator. For generating `time.Time` values, you can use the [jonboulle/clockwork](https://github.com/jonboulle/clockwork) package.

## Type system

### Sum types

Go doesn't have native support for sum types (discriminated unions). However, the pattern itself is quite useful and still can be achieved with a few caveats. 

```go
package user

type User interface {
	isUser()
}

type Customer struct {}
func (c Customer) isUser () {}

type Vendor struct {}
func (v Vendor) isUser () {}
```

This example shows a `User` interface with one sealed method and two structs implementing it. This approach ensures that only structs within the `user` package can implement the method. When combined with a [linter](https://github.com/alecthomas/go-check-sumtype) that checks for exhaustive switch case blocks, you effectively get sum types in Go.

## Testing

### Test with `GOMAXPROCS` set to 1

When testing code that involves concurrency, you might encounter a situation where tests consistently pass on your local machine but occasionally fail during CI/CD. One common reason for this is that CI/CD environments often allocate less than one CPU core to execute tests. As a result, certain operations you expect to complete in a specific order might actually execute later than anticipated, leading to unexpected behavior and test failures.

As a workaround, you can set `GOMAXPROCS` to `1` to help identify which assertions are failing in the test. Once you've pinpointed the issue, consider using `assert.Eventually` from the [testify/assert](github.com/stretchr/testify/assert) package. This function allows you to check conditions over an extended period rather than expecting immediate results, which is particularly useful for asynchronous operations.

### Use containers instead of mocks

When testing an implementation of an interface that interacts directly with external services (such as a database, cache, or message broker), it's best to use the corresponding container image instead of a mock, stub, or fake — whenever possible. Here's why:

- Mocks can behave differently from real service instances, leading to inaccurate or misleading test results.
- Mocks don’t perform real network communication, which might hide potential networking issues you'd encounter in production.
- To mimic the behavior of the actual service (e.g., SQL engine), you may end up reimplementing complex functionality—making your tests harder to maintain and potentially flawed.

In short, mocks typically fall short of accurately reflecting real-world conditions and move you further away from a production-like environment.

Tools like [testcontainers-go](https://github.com/testcontainers/testcontainers-go) and [dockertest](https://github.com/ory/dockertest) can help set up and manage lightweight, disposable containers during testing.
