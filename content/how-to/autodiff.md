---
title: "How to compute gradient (differentiation)"
date: 2019-10-29T20:07:07+01:00
draft: false
---

## Goal
Consider this simple equation:

$$ f(x,y,z) = ( x + y ) \times z $$

The goal of this article is to show you how Gorgonia can evaluate the gradient $\nabla f$ with its partial derivatives:

$$ \nabla f = [\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}] $$

### Explanation

Using the [chain rule](https://www.khanacademy.org/math/ap-calculus-ab/ab-differentiation-2-new/ab-3-1a/a/chain-rule-review), we can compute the gradient value at each step as illustrated here:

{{<mermaid align="left">}}
graph LR;
    x -->|$x=-2$<br>$\partial f/\partial x = -4$| add
    y -->|$y=5$<br>$\partial f/\partial y = -4$| add
    add(+) -->|$q=3$<br>$\partial f/\partial q = -4$| mul
    z -->|$z=-4$<br>$\partial f/\partial z = 3$| mul
    mul(*) -->|$f=-12$<br>$1$| f
{{< /mermaid >}}


{{% notice info %}}
For more info on the gradient computation, please read this [article from cs231n](http://cs231n.github.io/optimization-2/) from Stanford.
{{% /notice %}}

We will represent this equation into an [exprgraph](/reference/exprgraph) and see how to ask Gorgonia to compute the gradient.

When the computation is done, each node will hold a [dual value](/reference/dualvalue) that will contain both the actual value and the derivative wrt to x.

for example, considering the node x:

```go
var x *gorgonia.Node
```

Once Gorgonia has evaluated the exprgraph, it is possible to extract the value of `x` and the value of the gradient $\frac{\partial f}{\partial x}$ by calling:

```go
xValue := x.Value()    // -2
dfdx, _ := x.Grad()    // -4, please check for errors in proper code
```

Let's see how to do that.

## Creating the equation

First, let's create the [exprgraph](/reference/exprgraph) that represents the equation.

{{% notice info %}}
If you want more info on this part, please read the [hello world](/tutorials/hello-world/) tutorial.
{{% /notice %}}

```go
g := gorgonia.NewGraph()

var x, y, z *gorgonia.Node
var err error

// define the expression
x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
q, err := gorgonia.Add(x, y)
if err != nil {
    log.Fatal(err)
}
result, err := gorgonia.Mul(z, q)
if err != nil {
    log.Fatal(err)
}
```

And set some values:

```go
gorgonia.Let(x, -2.0)
gorgonia.Let(y, 5.0)
gorgonia.Let(z, -4.0)
```

### Getting The Gradients

There are two options to get the gradient:

* by using the [automatic differentiation](https://en.wikipedia.org/wiki/Automatic_differentiation) capability of the [LispMachine](/reference/lispmachine);
* by using the [symbolic differentiation](https://en.wikipedia.org/wiki/Computer_algebra) capability offered by Gorgonia;


#### Automatic Differentiation

Automatic differentiation is only possible with the [LispMachine](/reference/lispmachine).
By default, lispmachine performs forward mode and backwards mode execution.

Therefore, calling the RunAll method is enough to get the result.
```go
m := gorgonia.NewLispMachine(g)
defer m.Close()
if err = m.RunAll(); err != nil {
    log.fatal(err)
}
```

The values and gradients can now be extracted:

```go
fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
fmt.Printf("f(x,y,z) = %v\n", result.Value())

if xgrad, err := x.Grad(); err == nil {
    fmt.Printf("df/dx: %v\n", xgrad)
}
if ygrad, err := y.Grad(); err == nil {
    fmt.Printf("df/dy: %v\n", ygrad)
}
if xgrad, err := z.Grad(); err == nil {
    fmt.Printf("df/dx: %v\n", xgrad)
}
```


#### Symbolic Differentiation

Another option is to use symbolic differentiation.
Symbolic differentiation works by adding new nodes to the graphs. The new nodes represents holds the gradients with regards to to the nodes passed as argument.

To create those new nodes, we use the [Grad()](https://godoc.org/gorgonia.org/gorgonia#Grad) function.

Grad takes a scalar cost node and a list of with-regards-to, and returns the gradient

Consider the following code:
```go
var grads Nodes
if grads, err = Grad(result,z, x, y); err != nil {
    log.Fatal(err)
}
```

What this says is to compute the partial derivatives (gradients) with regards to `z`, `x` and `y`.



`grads` in an array of `[]*gorgonia.Node`, in the same order as the WRTs that are passed in:

* `grads[0]` = $\frac{\partial f}{\partial z}$
* `grads[1]` = $\frac{\partial f}{\partial x}$
* `grads[2]` = $\frac{\partial f}{\partial y}$

The gradient is compatible with both [TapeMachine](/reference/tapemachine) and [LispMachine](/reference/lispmachine). But TapeMachine is much faster.

```go
machine := gorgonia.NewTapeMachine(g)
defer machine.Close()
if err = machine.RunAll(); err != nil {
        log.Fatal(err)
}

fmt.Printf("result: %v\n", result.Value())
if zgrad, err := z.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", zgrad, grads[0].Value())
}

if xgrad, err := x.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", xgrad, grads[1].Value())
}

if ygrad, err := y.Grad(); err == nil {
        fmt.Printf("dz/dy: %v | %v\n", ygrad, grads[2].Value())
}
```

Note that you may access the partial derivatives in two ways:

1. By using the `.Grad()` method. e.g. for the gradient of `x` in the running example, use `x.Grad()`
2. By using the `.Value()` method of the gradient node. e.g. for the gradient of `x` in the running example, use `grads[1].Value()`.

The reason for having these two different ways of doing things comes down to suitability. When it is more meaningful to get values from the gradient nodes (e.g. you might want to compute the second derivative), then use the gradient nodes. But if you want a fast fetching of the gradient values, the `.Grad()` method might be most suitable. Ultimately it comes down to your taste.

## Full Code (Automatic Differentiation)

```go
func main() {
	g := gorgonia.NewGraph()

	var x, y, z *gorgonia.Node
	var err error

	// define the expression
	x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
	y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
	z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
	q, err := gorgonia.Add(x, y)
	if err != nil {
		log.Fatal(err)
	}
	result, err := gorgonia.Mul(z, q)
	if err != nil {
		log.Fatal(err)
	}

	// set initial values then run
	gorgonia.Let(x, -2.0)
	gorgonia.Let(y, 5.0)
	gorgonia.Let(z, -4.0)

	// by default, lispmachine performs forward mode and backwards mode execution
	m := gorgonia.NewLispMachine(g)
	defer m.Close()
	if err = m.RunAll(); err != nil {
		log.fatal(err)
	}

	fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
	fmt.Printf("f(x,y,z)=(x+y)*z\n")
	fmt.Printf("f(x,y,z) = %v\n", result.Value())

	if xgrad, err := x.Grad(); err == nil {
		fmt.Printf("df/dx: %v\n", xgrad)
	}

	if ygrad, err := y.Grad(); err == nil {
		fmt.Printf("df/dy: %v\n", ygrad)
	}
	if xgrad, err := z.Grad(); err == nil {
		fmt.Printf("df/dz: %v\n", xgrad)
	}
}
```

which gives:

```text
$ go run main.go
x=-2;y=5;z=-4
f(x,y,z)=(x+y)*z
f(x,y,z) = -12
df/dx: -4
df/dy: -4
df/dz: 3
```

## Full Code (Symbolic Differentiation)


```go
func main() {
	g := gorgonia.NewGraph()

	var x, y, z *gorgonia.Node
	var err error

	// define the expression
	x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
	y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
	z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
	q, err := gorgonia.Add(x, y)
	if err != nil {
		log.Fatal(err)
	}
	result, err := gorgonia.Mul(z, q)
	if err != nil {
		log.Fatal(err)
	}

	if grads, err = Grad(result,z, x, y); err != nil {
		log.Fatal(err)
	}

	// set initial values then run
	gorgonia.Let(x, -2.0)
	gorgonia.Let(y, 5.0)
	gorgonia.Let(z, -4.0)

	machine := gorgonia.NewTapeMachine(g)
	defer machine.Close()
	if err = machine.RunAll(); err != nil {
		log.Fatal(err)
	}

	fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
	fmt.Printf("f(x,y,z)=(x+y)*z\n")
	fmt.Printf("f(x,y,z) = %v\n", result.Value())

	if zgrad, err := z.Grad(); err == nil {
		fmt.Printf("dz/dx: %v | %v\n", zgrad, grads[0].Value())
	}

	if xgrad, err := x.Grad(); err == nil {
		fmt.Printf("dz/dx: %v | %v\n", xgrad, grads[1].Value())
	}

	if ygrad, err := y.Grad(); err == nil {
		fmt.Printf("dz/dy: %v | %v\n", ygrad, grads[2].Value())
	}
}
```

which gives:

```text
$ go run main.go
x=-2;y=5;z=-4
f(x,y,z)=(x+y)*z
f(x,y,z) = -12
df/dx: -4 | -4
df/dy: -4 | -4
df/dz: 3 | 3
```
