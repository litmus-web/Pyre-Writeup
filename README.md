# Pyre-Writeup

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

-- INSERT LATENCIES CHART ETC HERE -- 

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

#### Acceptable Limitations
After discussing with members of the web development community I have determined a suitable set of acceptable limitations given the timeframe of development.
- The server will only support the HTTP/1.x protocol (although the web socket protocol could be developed as an extension)
- The server will not support TLS/SSL encryption and handshaking.
- The framework will not provide inbuilt / pre-made middlewear.
- The framework will not support the legacy HTTP/1 Pipelining system.

#### Objectives
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

