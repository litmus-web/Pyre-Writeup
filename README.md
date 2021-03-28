# Pyre-Writeup

## Table of Contents

- [Analysis](https://github.com/Project-Dream-Weaver/Pyre-Writeup/blob/main/README.md#analysis)
- [Design](https://github.com/Project-Dream-Weaver/Pyre-Writeup/blob/main/README.md#documented-design)
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

### Basic Design Principles

#### HTTP Request request - response cycle
![image](https://user-images.githubusercontent.com/57491488/112757610-5c5e5900-8fe2-11eb-9bc8-4ff1086ef6f1.png)

1. The client sends the initial HTTP request with a socket connection, this looks something like this: 
```
GET /hello.htm HTTP/1.1
User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)
Host: www.tutorialspoint.com
Accept-Language: en-us
Accept-Encoding: gzip, deflate
Connection: Keep-Alive
```
2. The server process accepts the pending socket and begins reading data from the socket when able to, putting the data into a read buffer.
3. The parser then takes the read data and parses it into it's core components:
    - Method
    - Path
    - Protocol and Version
    - Headers
    - Body
4. Once the request has been parsed the resultant data is sent to the framework which handles all the logic and handling of that response.
5. The framework returns a structured response containing the relevant information
6. The response builder turns this structured response into the raw binary data and adds it to the writer buffer.
7. When the socket can be written to data is taken from the buffer and written to the socket.
8. The client receives this data and gets their response back.


#### ASGI Server Overview
![image](https://user-images.githubusercontent.com/57491488/112757585-46e92f00-8fe2-11eb-867c-3eff8cd074f6.png)


### Server Design

The server will be built off of the raw AsyncIO api for watching for socket ready-ness, I have chosen not to use the inbuilt Asyncio protocol system as it leads to several conflicts in design between Rust and Python which hurts both performance and code maintainability.

Instead I plan on creating a Rust replica of this system but with a few changes:
- The system will be poll based rather than callback based.
- Protocols will be made for each supported protocol and rotate by a single manager.
- The server will have control of a given reader and writer buffer which is shared to the protocols not passed by value.
- Clients will stay inside the Rust system, it will never be given to Python. Only file descriptor handling will be maanged by python.

#### Memory Management

Memory management is a key factor in my design as it can make or break both stability and performance, in order to proply make use of Rust's low level style I have came up with some set points:

##### Speed via bulk allocations
To improve performance Pyre will use bulk style allocation, meaning protocols will allocate their required memory maximum as soon as they're made rather than when they're needed, this cuts down memory allocations down to a fraction of what they would be at the cost of memory usage which brings me onto my next point.

##### Minimising memory usage
The server is opting to use more agressive memory usage in order to aid performance however, the this should be re-used for many clients rather than allocated once, used once and then removed as large allocations can take more time to allocate.

For this I plan on use the [slab crate](https://crates.io/crates/slab) to allow me to re-use allocated client handlers as well as [crossbeam](https://crates.io/crates/crossbeam) to provide me with a set of utils (in this case a array based queue) in order to give a queue of free clients to re-use allocations.

#### Utilities

Due to my mix of both Python and Rust exception handling can become awkward, because of this I plan on using Rust's macro system in order to make my code more DRY and convert Rust errors to Python errors in a cleaner system.

Macros are a method of build time code generation in rust, this can simplify and reduce the code base considerably in Rust.

##### Conversion Macro
```rust
macro_rules! conv_err {
    ( $e:expr ) => ( $e.map_err(|e| PyRuntimeError::new_err(format!("{}", e))) )
}
```

#### Server Flow
![image](https://user-images.githubusercontent.com/57491488/112757289-0c32c700-8fe1-11eb-990e-6793f7259f3e.png)

This flow chart shows the basic execution model of the Protocol (ignoring TCP keep alive handlers, receiver functions and processors) 

#### Parsing

One of the key parts of my server will be the parser, for the HTTP/1 protocol I will be using the [httparse crate](https://crates.io/crates/httparse) in order to provide a zero copy parser, this parser using the push style rather than a callback based system which is more suited to Rust's style.

Due to limtations however, this required me to pre-define an allocate a array of headers, this mean I am going to allow a maximum of 100 headers which is more than enough due to the average amount of headers being no more than 30 headers. This does create the issue however, that this can create bulky allocations which can be slow and in-efficient and also increase memory usage.

##### Fixing the Header issue

To get around the aformentioned issue with pre-defining header amounts I am going to be using unsafe Rust which forgoes the memory safety and constraints of the normal language, this is generally very dangerous to use and must be considered carefully, for this specific case however, we will be fine as we garentee that the un-initialised memory using `mem::unitialised()` will be initialised before we use it.

This solution fixes the issue of bulky allocations by faking the memory allocation in such a way to satify the Rust compiler and allow itself to be initialised when data is actually written to it, this means we are not actually allocating 100 headers instead we only allocate what we need.

##### Basic Parsing Design
```rust
// This should be fine as it is guaranteed to be initialised
// before we use it, just waiting for the ability to use
// MaybeUninit, till then here we are.
let mut headers: [Header<'_>; MAX_HEADERS] = unsafe {
    mem::uninitialized()
};

let body = buffer.clone();

let mut request = Request::new(&mut headers);
let status = conv_err!(request.parse(&body))?;

let len= if status.is_partial() {
    return Ok(())
} else {
    status.unwrap()
};
```

##### Parser Logic Flow
![image](https://user-images.githubusercontent.com/57491488/112762383-427b4100-8ff7-11eb-8fe3-9aed0495c856.png)


#### Flow Control
Flow control  is a essential part of my HTTP server as it is responsible for controlling data written to the socket buffer and data received from the socket, if we do not control the flow of data it could leave us vulnerable to response inject attacks potentially using more memory than we have causing the program or physical server itself to crash.

The flow control will be represented by a struct containing atomic bools to represent the state of flow and the transport object for interacting with the socket itself. Due to the nature of Rust’s programming ecosystem I have constructed a prototype in Python that considers how it will be re-implemented in Rust.

##### Python Prototype
```py
import asyncio

class FlowControl:
    def __init__(self, transport: asyncio.Protocol):
        self.transport = transport
        self.is_read_paused = False
        self.is_write_paused = False
        self.waiter = asyncio.Event()
    
    def pause_reading(self):
        if not self.is_read_paused:
            self.is_read_paused = True
            self.transport.pause_reading()
            
    def resume_reading(self):
        if self.is_read_paused:
            self.is_read_paused = False
            self.transport.resume_reading()
            
    def pause_writing(self):
        if not self.is_write_paused:
            self.is_write_paused = True
            self.waiter.set()
            
    def resume_writing(self):
        if self.is_write_paused:
            self.is_write_paused = False
            self.waiter.clear()
```

#### Protocol Switching

The ability to switch protocol is a key aspect of my project to allow for future expandability, this may prove to be a challenge in Rust due to it's immutable nature. However, I have managed to come up with a solution using Rust's Enums to register each targetted protocol e.g.

```rust
enum Protcols {
    H1,
    H2,
    WS,
}
```

These enums will help me choose which protocol to target with events from the main server handler, although this does not stop the limitation that I must create each protcol pre-emptively rather than lazily, although this should not be an issue for my investigation as I am only using one set protocol (HTTP/1 or H1 as I will call it).

##### Protocol Logic Flow
![image](https://user-images.githubusercontent.com/57491488/112757569-2faa4180-8fe2-11eb-8a46-83e012c6bac3.png)




