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

### Step #1: Clone de tutorial code repository

Clone the following

Let's imagine we are writing a client component that interacts with a REST API. One obvious dependency that we will have to mock in our unit tests is the HTTP client one. So to be able to mock this dependency, the first step is to write the code that uses the HTTP client to use an interface rather than to refer to a concrete type such as `http.Client`.

###

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





