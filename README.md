# gomock-tutorial
A quick and dirty Gomock tutorial to learn how to do unit testing in Go.

## Knowledge prerequisites

* Basic knowledge of Go programming
* Basic understanding of Go interfaces

## Software prerequisites

* Go (1.14)
* Git

## What is unit testing?

Unit testing is the practice of verifying the correctness of individual units or components of a software. The purpose is to validate that each unit of the software code performs as expected. A unit may be an individual function, method, procedure, module or object. Unit testing is performed by developers while coding the application. As a matter fact, Test-driven Development (TDD) stipulates that test must be written before the actual unit code. 

## What is mocking?

Mocking is a process used in unit testing to efficiently handle external dependencies. The purpose of mocking is to enable developers to focus on the code being tested and not on the behavior or state of external dependencies. This is done by replacing dependencies by objects that simulate the behavior of the real ones. These replacement objects generally fall in the following categories: 

* *Fakes*: Objects that are programmed to statically simulate a specific behavior and return specific values. For complex tests, the use of fakes requires a large number fakes to be written.
* *Stubs*: Objects that will return a specific result based on a specific set of inputs.
* *Mocks*: Sophisticated version of stubs that can be programmed to return specific values and to enforce specific aspects of how functions are called including:
    * The number of times they are called
    * The order in which they are called
    * The parameters that are passed to them

## Why Gomock?

There several mocking frameworks out there for Golang and all have their pros and cons. We chose Gomock because of its ease of use and the following key features:
* Automatic mock generation via a CLI
* Argument matchers
* Integration with Go tooling
* Powerful expectation API

## Tutorial

There is no better way to learn than to get your hands dirty, so let's dive right in!

### Step #0: Clone de tutorial code repository

Clone the `github.com/walmartdigital/gomock-tutorial-code` Git repository.

```
git clone git@github.com:walmartdigital/gomock-tutorial-code.git
```

### Step #1: Create an interface for the HTTP client

Take a look at the directory structure of the project:

```
.
├── README.md
├── go.mod
├── go.sum
├── main.go
└── pkg
    └── client
        └── client.go
```
Basically, the program consists of a `main` that creates an HTTP server which serves two routes `/monkeys` and `/dogs`. There is a client component whose code lives in the `client` package and that interacts with the HTTP server. The `client` package is the *unit* we will test throughout this tutorial.

From the root of the project, run the program:

```
▶ go run .                     
Hi there, I love monkeys!
Hi there, I love dogs!
```

As you can see from `pkg/client/client.go`, the code depends on the `github.com/go-resty/resty/v2` client library to interact with the HTTP server. This is rather impractical for doing unit testing as it requires us to set up a real HTTP server in our test code or to use some dirty hack involving monkey patching to replace the dependency in our test code.

```go
func ReadMessage(animal string) string {
	client := resty.New()
	resp, _ := client.R().Get(fmt.Sprintf("http://localhost:8080/%s", animal))
	return string(resp.Body())
}
```

To be able to mock this dependency cleanly, the first step is to replace the dependency on the concrete type `resty.Request` by a Go interface such as the following:

```go
type HTTPRequest interface {
    Get(url string) (*resty.Response, error)
}
```

In order for the program to run, you will have to refactor the `ReadMessage()` function to receive the `HTTPRequest` object as a parameter and pass the concrete object of type `resty.Request` from the `main` function.

```go
func ReadMessage(request HTTPRequest, animal string) string {
	resp, _ := request.Get(fmt.Sprintf("http://localhost:8080/%s", animal))
	return string(resp.Body())
}
```

```go
func main() {
	http.HandleFunc("/monkeys", monkeys)
	http.HandleFunc("/dogs", dogs)
	go http.ListenAndServe(":8080", nil)
	c := resty.New()
	fmt.Println(client.ReadMessage(c.R(), "monkeys"))
	fmt.Println(client.ReadMessage(c.R(), "dogs"))
}
```
From the root of the project, run the program again to see if the refactoring worked:

```
▶ go run .                     
Hi there, I love monkeys!
Hi there, I love dogs!
```
Looking good! Now, let's mock the `HTTPRequest` interface.

### Step #2: Create the mocks

The first step here is to install Gomock:

```
GO111MODULE=on go get github.com/golang/mock/mockgen@latest
```

Let's create the directory where we will maintain the `mocks` package:

```
mkdir pkg/mocks
```
Using the `mockgen` utility, let's generate the mocks for the `HTTPRequest` interface:

```
mockgen -source=pkg/client/client.go -destination=pkg/mocks/client.go -package=mocks
```
Using the `-source` option, we point it to the source file containing the target interface. The `-destination` argument tells `mockgen` where to store the generated code and the `-package` argument specifies in which package the mocks will be present.

As you can see, `mockgen` created the `pkg/mocks/client.go` file which contains the mocks for the `HTTPRequest` interface.

Now that we have generated the necessary mocks, we are ready to write tests that will use these mocks.

### Step #3: Write the tests

### Key concepts to cover in tutorial
#### How to generate mocks using mockgen
#### Using Expect() to expect function calls
#### Using Return() to control return values
#### Using setAttribute() to control the value of attributes passed by reference
#### Mention inOrder() to specify the order in which function should be called


### Additional resources
This tutorial was heavily-based on some very valuable material found on the Internet, including:

* https://www.telerik.com/products/mocking/unit-testing.aspx
* https://blog.codecentric.de/2019/07/gomock-vs-testify/
* https://www.guru99.com/unit-testing-guide.html





