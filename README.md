# Pyre-Writeup

## Analysis
### The problem
The current Python ecosystem boasts a wide variety of webservers and frameworks making it one of the key players in the ecosystem.
However, most servers are starting to run into the following issues:
- Slow in both throughput and latency
- Insecure
- Less robust 
- Stability at higher connections

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
- Improve server throughput by at least 30%.
- Improve server latency by at least 30%.
- Provide a upgraded interface which is built off of the HTTP spec.
- Provide a ASGI compatability layer to allow for existing framework intergration.
- Comply with the modern HTTP/1 spec.

#### Framework
- Provide a minimalistic, object oriented routing.
- Provide the ability to intergrate both local and global middlewear.
- Provide a sutable response framework for returning HTTP/1 complient responses to the server.
- Provide a implicit conversion system to convert url arguements into specified types.

#### Propesed Framework Design
```py
from pyre import framework
from pyre.framework import App, Request, responses


class MyEndpoint(framework.Blueprint):
    def __init__(self, app: App):
        self.app = app

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
