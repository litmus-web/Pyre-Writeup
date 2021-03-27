# Pyre-Writeup

## Table of Contents

- [Analysis]()
- [Design]()
- [Code Repo]()
- [Testing]()
- [Evaluation]()

## Analysis
### The problem
The current Python ecosystem boasts a wide variety of webservers and frameworks making it one of the key players in the ecosystem.
However, most servers are starting to run into the following issues:
- Slow in both throughput and latency
- Insecure
- Less robust 
- Lack stability at higher connections

Equally Python frameworks are also starting to suffer from the following issues:
- Very little diversity in the ecosystem
- Many incompatible changes
- Are all mainly copies of the original framework called `Flask` but with small modifications
- Provide a limited set of converters for url arguements.

In comparrison to the web systems being developed in other languages as the modern world progresses.

Until now the 'fastest' Python server has been a server called `uvicorn`, in reality however, uvicorn is still slow compared to other modern frameworks. On the other hand Python does offer what most other languages do not which is a set of universal, standardised interfaces for servers and frameworks to interact in order to provide a more modular design for the developer; This set of interfaces in themselves are becoming outdated or provide a meaningless abstraction over the standardised HTTP protocol.

![latencies](https://user-images.githubusercontent.com/57491488/112694270-f8208580-8e79-11eb-957e-9b488f9c9b0f.png)
![requests](https://user-images.githubusercontent.com/57491488/112694339-18e8db00-8e7a-11eb-8311-b1d7ca55bb4c.png)


### Proposed Solution
In my investigation I plan to develop a HTTP server and framework which improves the aformentioned points and acheive the following goals:

#### Server
- Be asyncronous and non-blocking by design.
- Improve server throughput by upto 40%.
- Improve server latency by upto 40%.
- Provide a upgraded interface which is built off of the HTTP spec.
- Provide a ASGI compatability layer to allow for existing framework intergration.
- Comply with the modern HTTP/1 spec.

##### What is significant about 40pct?
Using [TechEmpower Benchmarks](https://www.techempower.com/benchmarks/#section=data-r20&hw=ph&test=fortune&l=zijzen-sf) as a referance; Uvicorn currently benchmarks at around `70,000` requests per second throughput, a 40% increase in throughput would result in a increased throughput of around `30,000` requests / sec reducing the amount of workers needed to supply `70,000` requests per second from 16 workers to ~11 workers, this can produce a large amount of savings for a company upgrading from Uvicorn to Pyre due to the lower amount of hardware required to produce the same results.


#### Framework
- Be asyncronous and non-blocking by design.
- Provide a minimalistic, object oriented routing.
- Provide the ability to intergrate both local and global middlewear.
- Provide a sutable response framework for returning HTTP/1 complient responses to the server.
- Provide a implicit conversion system to convert url arguements into specified types.

#### Existing Framework Style(s)
```py
# main.py

from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
```

While the above design works nicely for a small design this does not scale as well when the codebase grows, Flask currently implements `blueprints` however, these prevent being able to back reference the app itself which can add extra limitations and code complexity when design the app with a shared state between endpoints.

#### Proposed Framework Style
```py
# blueprint.py

from pyre import framework
from pyre.framework import App, Request, responses


class MyEndpoint(framework.Blueprint):
    def __init__(self, app: App):
        self._app = app

    @framework.endpoint("/hello/{name:string}", methods=["GET"])
    async def on_get_hello(self, name: str):
        return f"Hello, {name}!"

    @framework.endpoint("/hello/{name:string}", methods=["POST"])
    async def on_post_hello(self, request: Request, name: str):
        body = await request.body.json()
        return responses.JSONResponse(
            {"message": f"Hello, {name}! You are aged: {body['age']}"},
            status=200,
        )
    

def setup(app: App):
    app.add_blueprint(MyEndpoint(app))
```
This style while being more bulky for small designs, provides a much more robust design for larger code bases being able to referance the web app itself while also being able to be stored in seperate files for diffrent sections.

### Acceptable Limitations
After discussing with members of the web development community I have determined a suitable set of acceptable limitations given the timeframe of development.
- The server will only support the HTTP/1.x protocol (although the web socket protocol could be developed as an extension)
- The server will not support TLS/SSL encryption and handshaking.
- The framework will not provide inbuilt / pre-made middlewear.
- The framework will not support the legacy HTTP/1 Pipelining system.

### Objectives
1. Provide a asyncronous HTTP server supporting the HTTP/1.x protocol.
2. Support both standard methods of writing the body to a HTTP request (fixed content length / chunked encoding).
3. Correctly handle socket errors like disconnects, interupts and EOFs.
4. Support both the ASGI (Asyncronous Server Gateway Interface) and custom PSGI (Pyre Server Gateway Interface) interfaces.
5. Provide pre-compiled Python wheels for ease of development for users on Linux as well as Windows x64 architecture.
6. Reduce latency by upto 40%.
7. Increase request throughput by upto 40%.
8. Provide both local and global middlewear handlers.
9. Provide sufficient request routing.
10. Provide sufficient arguement conversion between types.

## Documented Design

### General Design Summary
Now that I have laid out my objectives from my analysis i can now start to design my project. I plan to split the project into two parts; The server which will utilise a ASGI interface and PSGI interface and the framework itself which shall be a minimalistic micro-framework.

#### Server Summary
The server will be the thing to actually run the framework itself, I want to design this in such a way that I can be used for more than just this framework however making it more maintainable in the future.To achieve this, I am going to be using the ASGI interface which is a superset of the WSGI interface designed for Python web frameworks and server, it is designed to solve the following issue described in the Python PEP 333(3):

*“Python currently boasts a wide variety of web application frameworks, such as Zope, Quixote, Webware, SkunkWeb, PSO, and Twisted Web --to name just a few. This wide variety of choices can be a problem for new Python users, because generally speaking, their choice of web framework will limit their choice of usable web servers, and vice versa.” – PEP 333*

I plan to actually write the server in Rust and Python using the Rust crate PyO3 to produce Python binding, this will give native performance while still being used in Python making the server more suited for future growth should client find themselves managing more users than they originally thought on their web applications

#### Framework Summary
The framework will be built off the same ASGI interface that the server will, this also allows the framework to be ran on multiple types of servers e.g. Gunicorn, UWSGI, etc...

As laid out by the objectives in my analysis the framework needs to:
- Handling URL routing.
- Give access to local and global middlewears.
- Provide HTML, JSON and Plain Text responses.
- Provide a class-based view system with.
- Dynamic loading and unloading of views.
- Sessions.
- Cookie control.

### File Structure
My project file structure is largely determined by the requirments of Rust's package manager called `cargo` the Rust code will sit in 3 seperate folders called crates, each crate will have a diffrent purpose and allow the compiler to build each crate in parrallel to speed up compile times.

On top of these Rust crates I will define a Python module in order to provide the nessesary boiler plate and type linting for the end user / developer, this will be called `pyre` as it is the overall Python package name and is what will be referenced upon a import.

As well as the above i will also have a folder called `tests` and `benchmarks` which I will use to define my testing and benchmarking systems in order to aid in my design and building of my project.

##### Rust Crates
A crate will group related functionality together in a scope so the functionality is easy to share between multiple projects. All the functionality provided by the crate is accessible through the crate’s name.

Crates / Modules let us organize code within a crate into groups for readability and easy reuse. Modules also control the privacy of items, which is whether an item can be used by outside code (public) or is an internal implementation detail and not available for outside use (private).

#### Resulting File Structure
![Untitled Document(1)](https://user-images.githubusercontent.com/57491488/112734736-7bf07580-8f3f-11eb-9789-2f4c7e3cbae7.png)

### Code styles
For my investigation I am using multiple programming languages; Python which is a object orientated, high level language, and Rust a low level systems programming language.

For each of my relevant programming languages I plan on following the standardised style guide. 
In Python's case I am following the PEP8 style guide meaning:
`snake_case` - For variables, functions, etc...
`CONSTANT_CASE` - For explicitly declaring constants.
`PascalCase` - For defining classes and types.

A leading `_` defines a private attribute.
A leading `__` defines a protected attribute.

For Rust I am following the standard coding style which is virtually identical to Python's PEP8 execpt that `PascalCase` is used on structs, traits and types. Rust does also not have the concept of dunder methods for inbuilts and special methods, in this case a leading `_` means a declaration is not used. 

#### Example Python Style
```py
FOO = 123
BAR = 321

class FooBarMaker:
    @staticmethod
    def make_foo_bar() -> int:
        return FOO + BAR

if __name__ == "__main__2:
    result = FooBarMaker.make_foo_bar()
    print(f"got {result}")
```

#### Example Rust Style
```rust
const FOO: usize = 123;
const BAR: usize = 321;

struct FooBarMaker;

impl FooBarMaker {
    fn make_foo_bar() -> usize {
        FOO + BAR
    }
}

fn main() {
    let result = FooBarMaker::make_foo_bar();
    println!("got {}", result);
}
```



