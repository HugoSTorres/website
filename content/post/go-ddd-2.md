+++
Description = "Following up on feedback from the last post"
Tags = ["go", "golang", "ddd", "domain driven design"]
date = "2017-09-17T19:56:55-04:00"
title = "Let's Talk Some More About Cars"

+++

There’s been a lot of great feedback on my last post about "Domain Driven Design."

Here’s the first bit I got from the golang-nuts group:

> Examples of what you call "factories" are right there in the go-to documents for Go beginners.

It’s absolutely true. It’s something I’ve been doing ever since I first started writing Go, because of those beginner documents. You can find more info on that here: https://golang.org/doc/effective_go.html#composite_literals

In general, a lot of what I wrote about has been stuff I’ve picked up in one form or another as I’ve gone honing my programming skills, and to be fair, "Domain Driven Design" has a lot more to it than what I wrote up in that post. What’s valuable is the way all the concepts come together, not just the ones I wrote about.

Another bit came from a conversation with u/DenzelM on r/golang.

The `Car` struct could be better. It should look like this:

```
// Car is the root object of our aggregate.
type Car struct {
	id string
	Color string
	wheels []wheel
}

// wheel is within our aggregate, so we don't let anybody
// access it from the outside. Each wheel must be accessed
// through a car.
type wheel struct {
	Diameter int
	Color string
}

// Wheels exposes the Car's wheels, while keeping the field
// private, ensuring the invariant can never be broken.
func (c Car) Wheels() []wheel {
    return c.wheels
}
```

This way, whatever business logic ties together a `Car` and its `wheels` can’t be undone.

This does mean that if you wish to expose the `wheels` in JSON, you would have to implement the appropriate interfaces yourself. This is because since the fields are private, the `json` package can never see them, so they will always be (un)marshaled into their zero values.

Here's what "Effective Go" has to say about getters and setters:

> There's nothing wrong with providing getters and setters yourself, and it's often appropriate to do so ...

So thanks to everybody who left feedback. It’s been great and has helped me learn even more.
