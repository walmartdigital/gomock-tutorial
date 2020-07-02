# gomock-tutorial
A quick and dirty Gomock tutorial to learn how to mock in Go using the Gomock framework.

## Knowledge prerequisites

* Basic knowledge of Go programming
* Basic understanding of Go interfaces

## Software prerequisites

* Go (1.14)
* Git

## What is unit testing?

Unit testing is the practice of verifying the correctness of individual units or components of a software. The purpose is to validate that each unit of the software code performs as expected. A unit may be an individual function, method, procedure, module or object. Unit testing is performed by developers while coding the application. As a matter fact, Test-driven Development (TDD) stipulates that tests must be written before the actual unit code. 

## What is mocking?

Mocking is a process used in unit testing to efficiently handle external dependencies. The purpose of mocking is to enable developers to focus on the code being tested and not on the behavior or state of external dependencies. This is done by replacing dependencies by objects that simulate the behavior of the real ones. These replacement objects generally fall in the following categories: 

* *Fakes*: Objects that are programmed to statically simulate a specific behavior and return specific values. For complex tests, the use of fakes requires a large number fakes to be written.
* *Stubs*: Objects that will return a specific result based on a specific set of inputs.
* *Mocks*: Sophisticated version of stubs that can be programmed to return specific values and to enforce specific aspects of how functions are called including:
    * The number of times they are called
    * The order in which they are called
    * The parameters that are passed to them

## Why Gomock?

There are several mocking frameworks out there for Golang and all have their pros and cons. We chose Gomock because of its ease of use and the following key features:
* Automatic mock generation via a CLI (`mockgen`)
* Argument matchers
* Integration with Go tooling
* Powerful expectation API

## Tutorial

There is no better way to learn than to get your hands dirty, so let's dive right in!

### Step #0: Clone de tutorial code repository

Clone the `github.com/walmartdigital/gomock-tutorial-code` Git repository.

```
git clone https://github.com/walmartdigital/gomock-tutorial-code.git
```
**Note that there are instructions throughout the tutorial indicating which Git tag to use in order to start each example/step from a clean state.**

### Step #1: Create an interface for the HTTP client

```
git checkout step-1
```

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

As you can see from `pkg/client/client.go`, the code depends on the `github.com/go-resty/resty/v2` client library to interact with the HTTP server. This is rather impractical for doing unit testing as it requires us to set up a real HTTP server in our test code or to use some dirty hack involving *monkey patching* to replace the dependency in our test code.

```go
func ReadMessage(animal string) string {
	client := resty.New()
	resp, _ := client.R().Get(fmt.Sprintf("http://localhost:8080/%s", animal))
	return string(resp.Body())
}
```
When it comes to mocking **in Go**, it is best to try and group all of the functionality you want to control under the same interface. Additionally, we want to use *Dependency Injection (DI)* to be able to dynamically specify which implementation of the interface we want to use. That is, we want to be able to specify from the calling code whether we want to use a real implementatio or a mock. To achieve that, we refactor the `client` package as follows:

```go
package client

import (
	"fmt"
)

// HTTPClient ...
type HTTPClient interface {
	Get(url string) (int, []byte, error)
}

// HTTPClientFactory ...
type HTTPClientFactory interface {
	Create() HTTPClient
}

// ZooClient ...
type ZooClient struct {
	client HTTPClient
}

// NewZooClient ...
func NewZooClient(factory HTTPClientFactory) *ZooClient {
	client := ZooClient{
		client: factory.Create(),
	}
	return &client
}

// ReadMessage ...
func (z *ZooClient) ReadMessage(animal string) string {
	_, body, _ := z.client.Get(fmt.Sprintf("http://localhost:8080/%s", animal))
	return string(body)
}
```
Here, we specify the `HTTPClient` interface which is clean and simple, hereby removing the need to manipulate `resty`-specific data structures such as the `*resty.Request` and `*resty.Response` as we previously did. The `HTTPClient` interface specifies a `Get()` function which takes a URL as argument and returns an HTTP status code, the response body and an error.

Additionally, we create the `ZooClient` wrapper that allows us to achieve dependency injection via the `NewZooClient()` *constructor* which accepts an `HTTPClientFactory` object whose behavior we can specify from the calling context.

As a result of the above changes, we also need to refactor the `main` package:

```go
// RestyClient ...
type RestyClient struct {
	client *resty.Client
}

// NewRestyClient ...
func NewRestyClient() *RestyClient {
	r := RestyClient{
		client: resty.New(),
	}
	return &r
}

// Get ...
func (r RestyClient) Get(url string) (int, []byte, error) {
	resp, err := r.client.R().Get(url)
	body := resp.Body()
	return resp.StatusCode(), body, err
}

// RestyClientFactory ...
type RestyClientFactory struct{}

// Create ...
func (f RestyClientFactory) Create() client.HTTPClient {
	r := NewRestyClient()
	return *r
}

func main() {
	http.HandleFunc("/monkeys", monkeys)
	http.HandleFunc("/dogs", dogs)
	go http.ListenAndServe(":8080", nil)

	zoo := client.NewZooClient(RestyClientFactory{})
	fmt.Println(zoo.ReadMessage("monkeys"))
	fmt.Println(zoo.ReadMessage("dogs"))
}
```
We created the `RestyClient` type (which implements the `HTTPClient` interface) and encapsulates all of the `resty` implementation-specific code. We also created the `RestyClientFactory` which we pass to the `NewZooClient()` function when creating the client. 

Let's check if our program still works. From the root of the project, run the program again to see if the refactoring worked:

```
▶ go run .                     
Hi there, I love monkeys!
Hi there, I love dogs!
```
Looking good! Now, let's move on to mocking the `HTTPClient` interface.

### Step #2: Create the mocks

```
git checkout step-2
```

The first step here is to install Gomock:

```
GO111MODULE=on go get github.com/golang/mock/mockgen@latest
```
Once Gomock is installed, we use the `mockgen` utility to generate the mocks for the `HTTPClient` interface:

```
mockgen -source=pkg/client/client.go -destination=pkg/mocks/client.go -package=mocks
```
Using the `-source` option, we point it to the source file containing the target interface. The `-destination` argument tells `mockgen` where to store the generated code and the `-package` argument specifies what package the mocks should be part of.

As you can see, `mockgen` created the `pkg/mocks/client.go` file which contains the mock for the `HTTPClient` interface called `MockHTTPClient` as well as the function to create a new mock `NewMockHTTPClient()`. It also generated similar code for the `HTTPClientFactory` interface.

Now that we have created the necessary mocks, we are ready to write tests that will use these mocks.

### Step #3: Write the tests

```
git checkout step-3
```

Let's a create the `pkg/client/client_test.go` test file for the client package:

```go
package client_test

import (
	"testing"

	"github.com/golang/mock/gomock"
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/walmartdigital/gomock-tutorial-code/pkg/client"
	"github.com/walmartdigital/gomock-tutorial-code/pkg/mocks"
)

var ctrl *gomock.Controller

func TestAll(t *testing.T) {
	ctrl = gomock.NewController(t)
	defer ctrl.Finish()

	RegisterFailHandler(Fail)
	RunSpecs(t, "Client tests")
}

var _ = Describe("Read message", func() {
	var (
		fakeHTTPClientFactory *mocks.MockHTTPClientFactory
		zooClient             *client.ZooClient
	)

	BeforeEach(func() {
		fakeHTTPClientFactory = mocks.NewMockHTTPClientFactory(ctrl)
		zooClient = client.NewZooClient(fakeHTTPClientFactory)
	})

	It("should read a message from the server", func() {
		msg := zooClient.ReadMessage("dogs")
		Expect(msg).To(Equal("Hi there, I love dogs!"))
	})
})
```
Here, we set up a simple test to exercise the *happy path* of our program. We create a `MockHTTPClientFactory` (which was generated by `mockgen`) and we pass it to the `NewZooClient()` function to create the client.

```
git checkout step-3a
```

Let's run the test and see what happens! From the `pkg/client` folder, run `go test`:

```
▶ go test
Running Suite: Client tests
===========================
Random Seed: 1593273859
Will run 1 of 1 specs

--- FAIL: TestAll (0.00s)
    client.go:25: Unexpected call to *mocks.MockHTTPClientFactory.Create([]) at /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/mocks/client.go:78 because: there are no expected calls of the method "Create" for that receiver
FAIL
exit status 1
FAIL    github.com/walmartdigital/gomock-tutorial-code/pkg/client       0.195s
```
We are getting an error telling us that there was an unexpected call to the `*mocks.MockHTTPClientFactory.Create([])` function which is called by the `client.NewZooClient()` call in our test. 

[Rolling drums...]

### Please meet `gomock`'s expectation API!!! :fireworks:

What we need to do fix this test is to use the `*mocks.MockHTTPClientFactory.EXPECT()` function to instruct `gomock` how we expect the `Create()` function to behave. Therefore, we modify the test as follows:

```go
var _ = Describe("Read message", func() {
	var (
		fakeHTTPClient        *mocks.MockHTTPClient
		fakeHTTPClientFactory *mocks.MockHTTPClientFactory
		zooClient             *client.ZooClient
	)

	BeforeEach(func() {
		fakeHTTPClient = mocks.NewMockHTTPClient(ctrl)
        fakeHTTPClientFactory = mocks.NewMockHTTPClientFactory(ctrl)
        
		fakeHTTPClientFactory.EXPECT().Create().Return(
			fakeHTTPClient,
        ).Times(1)
        
		zooClient = client.NewZooClient(fakeHTTPClientFactory)
	})

	It("should read a message from the server", func() {
		msg := zooClient.ReadMessage("dogs")
		Expect(msg).To(Equal("Hi there, I love dogs!"))
	})
})
```
Here we instruct `gomock` that we expect the `MockHTTPClientFactory.Create()` function to be called **exactly once**, with no argument and to return `fakeHTTPClient`.

```
git checkout step-3b
```

Let's run the test again and see what happens:

```
▶ go test
Running Suite: Client tests
===========================
Random Seed: 1593274259
Will run 1 of 1 specs

--- FAIL: TestAll (0.00s)
    client.go:32: Unexpected call to *mocks.MockHTTPClient.Get([http://localhost:8080/dogs]) at /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/mocks/client.go:39 because: there are no expected calls of the method "Get" for that receiver
FAIL
exit status 1
FAIL    github.com/walmartdigital/gomock-tutorial-code/pkg/client       0.211s
```
Oops, it failed again... but for another reason. The error we're getting now is that there was an unexpected call to `*mocks.MockHTTPClient.Get()`. Let's fix that by adding the following `EXPECT()` call:

```go
fakeHTTPClient.EXPECT().Get("http://localhost:8080/dogs").Return(
    200,
    []byte("Hi there, I love dogs!"),
    nil,
).Times(1)
msg := zooClient.ReadMessage("dogs")
Expect(msg).To(Equal("Hi there, I love dogs!"))
```
Here we instruct `gomock` to expect `Get()` to be called **once** with the `"http://localhost:8080/dogs"` URL and to return:
* The `200` HTTP response code
* A `byte` arrary corresponding to `"Hi there, I love dogs!"`
* A nil `error`

```
git checkout step-3c
```

Let's run the test again and see what happens!

```
▶ go test
Running Suite: Client tests
===========================
Random Seed: 1593274894
Will run 1 of 1 specs

•
Ran 1 of 1 Specs in 0.000 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 0 Skipped
PASS
ok      github.com/walmartdigital/gomock-tutorial-code/pkg/client       0.225s
```

It worked! We just created our first test mocking our HTTP client dependency and using the `gomock` expectation API. Let's add some more tests!

### Step #4: Add more test cases

```
git checkout step-4
```

So far, we've only written a test to verify the most basic behavior of our API. Let's use the `gomock` API to write some interesting cases such as when the server returns a non-200 response code or when a connection error occurs.

```go
It("should answer that it doesn't know the provided type of animals", func() {
    fakeHTTPClient.EXPECT().Get("http://localhost:8080/elephants").Return(
        404,
        []byte("Not found"),
        nil,
    ).Times(1)
    msg := zooClient.ReadMessage("elephants")
    Expect(msg).To(Equal("Hi there, what is an elephant!"))
})

It("should answer that it doesn't know the provided type of animals", func() {
    fakeHTTPClient.EXPECT().Get("http://localhost:8080/dogs").Return(
        -1,
        nil,
        errors.New("Could not connect to server"),
    ).Times(1)
    msg := zooClient.ReadMessage("dogs")
    Expect(msg).To(Equal("Hi there, the zoo is closed!"))
})
```
```
git checkout step-4a
```
Running the tests again yields the following output:
```
▶ go test
Running Suite: Client tests
===========================
Random Seed: 1593283939
Will run 3 of 3 specs

•
------------------------------
• Failure [0.000 seconds]
Read message
/Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:24
  should answer that it doesn't know the provided type of animals [It]
  /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:50

  Expected
      <string>: Not found
  to equal
      <string>: Hi there, what is an elephant!

  /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:57
------------------------------
• Failure [0.000 seconds]
Read message
/Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:24
  should answer that it doesn't know the provided type of animals [It]
  /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:60

  Expected
      <string>: 
  to equal
      <string>: Hi there, the zoo is closed!

  /Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:67
------------------------------


Summarizing 2 Failures:

[Fail] Read message [It] should answer that it doesn't know the provided type of animals 
/Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:57

[Fail] Read message [It] should answer that it doesn't know the provided type of animals 
/Users/vn0q31j/code/underworld/gomock-tutorial-code/pkg/client/client_test.go:67

Ran 3 of 3 Specs in 0.001 seconds
FAIL! -- 1 Passed | 2 Failed | 0 Pending | 0 Skipped
--- FAIL: TestAll (0.00s)
FAIL
exit status 1
FAIL    github.com/walmartdigital/gomock-tutorial-code/pkg/client       0.235s
```

Finally we modify the expect in the tests.

```go
It("should answer that it doesn't know the provided type of animals", func() {
  fakeHTTPClient.EXPECT().Get("http://localhost:8080/elephants").Return(
    404,
    []byte("Not found"),
    nil,
    ).Times(1)
  msg := zooClient.ReadMessage("elephants")
  Expect(msg).To(Equal("Not found"))
})

It("should answer that it doesn't know the provided type of animals", func() {
  fakeHTTPClient.EXPECT().Get("http://localhost:8080/dogs").Return(
    -1,
    nil,
    errors.New("Could not connect to server"),
    ).Times(1)
    msg := zooClient.ReadMessage("dogs")
    Expect(msg).To(Equal(""))
})
```
```
git checkout step-4b
```
Running the tests again yields the following output:
```
▶ go test
Running Suite: Client tests
===========================
Random Seed: 1593702832
Will run 3 of 3 specs

•••
Ran 3 of 3 Specs in 0.000 seconds
SUCCESS! -- 3 Passed | 0 Failed | 0 Pending | 0 Skipped
PASS
ok      github.com/walmartdigital/gomock-tutorial-code/pkg/client       0.025s
```

### Recap

Hopefully, this quick tutorial gave a you sense of how to do mocking with Gomock and how it might impact the design of your interfaces. Gomock's API is quite intuitive and powerful. Some of the features that were not covered in this tutorial include (but are not limited to) the ability to control the values of parameters passed by reference with the `gomock.setAttribute()` function or to specify in which order functions should be called via the `gomock.InOrder()` function. For more information, refer to the Gomock Github repository: https://github.com/golang/mock.

### Additional resources
This tutorial is heavily-based on some very valuable material found on the Internet, including:

* https://www.telerik.com/products/mocking/unit-testing.aspx
* https://blog.codecentric.de/2019/07/gomock-vs-testify/
* https://www.guru99.com/unit-testing-guide.html





