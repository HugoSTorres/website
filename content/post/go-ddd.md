+++
Description = "Bringing structure to Go's freedom"
Tags = ["go", "golang", "ddd", "domain driven design"]
date = "2017-08-18T00:23:08-04:00"
title = "Domain Driven Design and Go"

+++

Go is notoriously tricky when it comes to software design because of the amount of freedom it gives you.

Recently I’ve been reading “Domain Driven Design” by Eric Evans. It's been a breath of fresh air.

Here are some of the concepts in Domain Driven Design and how they can be used to help guide design decisions when writing Go programs.

## Aggregates

Aggregates are just logical boundaries around different associated types.

The important thing about an aggregate is that anything inside the boundary is only accessible from a value of a specific type, called the root object. All other types within the aggregate must be accessed through the root object.

This makes a package’s API much easier to reason about. Instead of directly dealing with all its different types and their respective associations, the consumer only has to worry about one or a few types. This makes it easier for the person writing the code consuming the package to think about the business logic instead of the nitty gritty details.

```
package auto

// Car is the root object of our aggregate.
type Car struct {
	ID string
	Color string
	Wheels []wheel
}

// wheel is within our aggregate, so we don't let anybody
// access it from the outside. Each wheel must be accessed
// through a car.
type wheel struct {
	Diameter int
	Color string
}
```

In the above example, all a consumer has to worry about is a whole `Car`. This leaves the consumer free to deal with business logic related to `Cars` and their `Wheels` as a whole, without having to worry about the logic in between the `Cars` and their `Wheels`.

## Factories

Factories are how you create new values of aggregate types. They are responsible for creating an aggregate value in a valid default state.

```
package auto

// types declared up here

// NewCar is a factory that returns a Car, initialized in a useful new state.
func NewCar(color, wheelColor string, wheelDiameter int) Car {
	return Car{
		ID: guid.New(),
		Color: color,
		Wheels: []wheel{
			wheel{Diameter: wheelDiameter, Color: wheelColor},
			wheel{Diameter: wheelDiameter, Color: wheelColor},
			wheel{Diameter: wheelDiameter, Color: wheelColor},
			wheel{Diameter: wheelDiameter, Color: wheelColor},
		},
	}
}
```

Here, the factory returns a completely initialized `Car` object, with all the other types in the aggregate initialized and tied to it. It would be invalid for a `Car` to be created without wheels, so the factory function handles this for us.

The alternative to this would be for the client code to tie the `Car` and its `Wheels` together. This leads so something like the following.

```
package auto

type Car struct {
	ID string
	Color string
	Wheels []Wheel
}

type Wheel struct {
	Diameter int
	Color string
}

// ========
package main

import "auto"

func main() {
    car := auto.Car{
        ID: guid.New(),
		Color: color,
		Wheels: []auto.Wheel{
			auto.Wheel{Diameter: wheelDiameter, Color: wheelColor},
			auto.Wheel{Diameter: wheelDiameter, Color: wheelColor},
			auto.Wheel{Diameter: wheelDiameter, Color: wheelColor},
			auto.Wheel{Diameter: wheelDiameter, Color: wheelColor},
		},
    }
}
```

This is messier for a couple of reasons.

First, the API exposed by the `auto` package is bigger than it has to be. To quote Rob Pike, less API is always a good thing.

Second, `main` now has to worry about `Wheels` and how they associate to a `Car`. This is a trivial example, but there could be other business logic around this, like the maximum diameter of a `Wheel` in relation to a specific model of `Car`.

We could just add the factory function to the `auto` package in this example as well, but now the consumer still has the option of getting `Wheels` by themselves and circumventing the business logic that goes into associating `Wheels` with a `Car`.

## Repositories

The idea of the repository is that it abstracts away all the technical details about the database. As far as the consuming code is concerned, the objects are all in memory.

The key thing about a repository is that it only operates on an aggregate’s root object. In our `Car` example, the `Wheel` type doesn’t get a repository. Everything to do with `Wheels` in the database is handled internally by the `Car` repository itself.

This keeps the client code free from having to worry about tying associations together. Instead, it can focus on the business logic.

Repositories are easily implemented in Go using interfaces.

Here's an example `Car` repository.

```
type CarStore interface {
	GetCar(id string) (Car, error)
	CreateCar(Car) (Car, error)
	UpdateCar(Car) (Car, error)
	DeleteCar(id string) (Car, error)
}
```

This is great for testing.

In the tests for the `auto` package, an implementation of `CarStore` can be created that's backed by a `map` of some kind. All the objects could be held in memory, making testing easier by removing an external dependency and eliminating the performance hit of a network request to the database.

The `auto` package can have an implementation backed by a real database as well, for use in the actual application.

In both cases, the consuming code (whether tests or actual application code) only sees an abstract representation of the object store. It doesn't actually know where those objects are stored, nor should it care.

## Packages

Packages are probably the hardest thing to organize in any Go project. It's very easy to get into situations where you have circular dependencies without taking the time to think about how to organize the code.

Fortunately, Domain Driven Design offers some tips on organizing packages.

Packages should be organized around specific concepts in the problem's domain. Each package should contain one or more highly cohesive aggregates. Different packages should have aggregates that are very loosely coupled to each other.

For example, in our `auto` example, if we were dealing with a domain of vehicles, we could have an `auto` package for automobiles like cars and trucks. We could also have a `watercraft` package, for things like boats, that are very different than automobiles even though they are part of the same domain.
