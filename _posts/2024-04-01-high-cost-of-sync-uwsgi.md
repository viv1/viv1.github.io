---
layout: post
title: Python on the web - High cost of synchronous uWSGI
date: 2024-04-01 12:30
category: Web Technologies
tags: ["uwsgi", "wsgi", "python", "web", "web technology", "GIL", "asgi", "asyncio"]
description: We talk about how the traditional synchronous web server protocol can be disadvantageous.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Python on Web](#python-on-web)
- [How uWSGI works](#how-uwsgi-works)
- [Experiment](#experiment)
   * [Setup](#setup)
   * [Run the tests](#run-the-tests)
   * [Results](#results)
- [Conclusion](#conclusion)
- [References and Further Reading](#references-and-further-reading)

<!-- TOC end -->



<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

`Python` is a popular choice for building programming due to its simplicity, readability, and vast ecosystem. Extending python as a webserver in a production environment is a little tricky due to the python [`GIL`](https://en.wikipedia.org/wiki/Global_interpreter_lock){:target="_blank"}. It basically renders most CPU bound multithreading pointless. We look into a popular option used widely and when and how it fails.

<!-- TOC --><a href="#" name="python-on-web"></a>
## Python on Web

`Python interpreter` is basically an instance of our python application. We need a bridge between `Python interpreter` and the `web server` to allow it to serve web requests. There are couple of popular standards for this bridge. We have `WSGI` and the latest `ASGI` standard. `ASGI` is a newer standard more suited for the latest async capabilities of python. Whereas `WSGI` is more suite for the traditional synchronous python.

In this article, we are going to focus on how CPU bound tasks can lead to terribel user experience with `WSGI`.

>[`WSGI Standard`](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface){:target="_blank"} helps us in extending python support to the web, which makes creating web applications using python possible.
One popular implementation in this standard is the [`uWSGI`](https://uwsgi-docs.readthedocs.io/en/latest/){:target="_blank"}.
{:.prompt-info}


<!-- TOC --><a href="#" name="how-uwsgi-works"></a>
## How uWSGI works

When we start a `uWSGI` instance for our `Python application`, it creates a `worker process` that runs an instance of the `Python interpreter` with our application code imported. And if we have `x` number of `worker processes` defined, it will `fork` until we have `x` number of interpreters running.


>We talk about `uWSGI workers` as processes, but `uWSGI workers` can be ***thread*** based as well. However they hardly ever work without issues for any python web application. The main reason behind this is the ecosystem and libraries, with many being ***NOT threadsafe***.
{:.prompt-info}

 `uWSGI` handles incoming HTTP requests and forward them to one of the available worker processes. Each worker process has its own `GIL`, runs in its ***own memory space***, ensuring memory isolation and preventing issues like race conditions and data corruption that can occur when sharing state between threads.

<!-- TOC --><a href="#" name="experiment"></a>
## Experiment

With our concepts clear, let's do a little experiment. We will see how incoming requests won't be served when the `uWSGI` workers are not available.

<!-- TOC --><a href="#" name="setup"></a>
### Setup

We will create a `server app` which will handle incoming requests. It is served by `uWSGI` with `5 worker processes`. It will take `10s` to respond to each request.

We will create a `client app` while will make severa (`20`) calls in parallel to the server at once. 

> You can directly get the code from the [repository](https://github.com/viv1/blog-code-examples/tree/main/uWSGI_blog){:target="_blank"}.
{:.prompt-info}

- Write a simple python application that return "true". Call it `app_main.py`

```python
import time

def long_running_task():
    time.sleep(10)
    return "true"

def application(env, start_response):
    query_string = env.get('QUERY_STRING', '')

    query_params = {}
    for param in query_string.split('&'):
        key, value = param.split('=')
        query_params[key] = value

    # Get the 'param' value from the query parameters
    param_value = query_params.get('param', 'No param value provided')

    print(f"Accept incoming request: {param_value}")
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [long_running_task().encode()]

```

- Create `Dockerfile` to do platform & OS independent setup for `uWSGI` (Assuming you have docker installed, otherwise you can install via [Docker website](https://www.docker.com/get-started/){:target="_blank"})

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9

# Set the working directory in the container
WORKDIR /app

# Install uWSGI and vim
RUN apt-get update && apt-get install -y build-essential vim
RUN pip install uwsgi

# Copy the app.py file into the container at /app
COPY app_main.py /app

# Make port 8000 available to the world outside this container
EXPOSE 8000

# Define environment variable
ENV NAME World

# Run uWSGI
CMD ["uwsgi", "--http", "0.0.0.0:8000", "--wsgi-file", "app_main.py", "--master", "--processes", "5"]

```

- Create a client that will make calls in parallel to the web server. Call it `client.py`

```python
import requests
from concurrent.futures import ThreadPoolExecutor

def send_request(query_param):
    print(f"Accept request: {query_param}")
    url = f'http://localhost:8000?param={query_param}'
    response = requests.get(url)
    print(f"Response: {response.text}")

def send_requests():
    with ThreadPoolExecutor(max_workers=20) as executor:
        futures = [executor.submit(send_request, f'Request_{i}') for i in range(20)]

if __name__ == "__main__":
    send_requests()

```

<!-- TOC --><a href="#" name="run-the-tests"></a>
### Run the tests

- In one terminal, run the following to start our `uWSGI` server:
```bash
docker build -t blog-uwsgi -f Dockerfile .
docker run -p 8000:8000 blog-uwsgi
```

- In another terminal, run the following to setup client app:
```bash
python3 -m venv venv
source venv/bin/activate 
pip install uwsgi
pip install requests
```

- Now, run the client app:

```bash
python client.py
```

<!-- TOC --><a href="#" name="results"></a>
### Results

This is my output:

{%
  include embed/video.html
  src='../video_assets/uwsgi_blog.mp4'
  types='mp4'
  title='Demo video'
  autoplay=true
  loop=true
  muted=true
%}

```bash

*** uWSGI is running in multiple interpreter mode ***
spawned uWSGI master process (pid: 1)
spawned uWSGI worker 1 (pid: 6, cores: 1)
spawned uWSGI worker 2 (pid: 7, cores: 1)
spawned uWSGI worker 3 (pid: 8, cores: 1)
spawned uWSGI worker 4 (pid: 9, cores: 1)
spawned uWSGI worker 5 (pid: 10, cores: 1)
spawned uWSGI http 1 (pid: 11)
Accept incoming request: Request_6
Accept incoming request: Request_4
Accept incoming request: Request_0
Accept incoming request: Request_1
Accept incoming request: Request_7
[pid: 8|app: 0|req: 1/1] 192.168.65.1 () {32 vars in 405 bytes} [Thu Apr 11 05:04:21 2024] GET /?param=Request_0 => generated 4 bytes in 10005 msecs (HTTP/1.1 200) 1 headers in 45 bytes (1 switches on core 0)
Accept incoming request: Request_9
[pid: 6|app: 0|req: 1/2] 192.168.65.1 () {32 vars in 405 bytes} [Thu Apr 11 05:04:21 2024] GET /?param=Request_4 => generated 4 bytes in 10006 msecs (HTTP/1.1 200) 1 headers in 45 bytes (1 switches on core 0)
Accept incoming request: Request_10
[pid: 7|app: 0|req: 1/3] 192.168.65.1 () {32 vars in 405 bytes} [Thu Apr 11 05:04:21 2024] GET /?param=Request_7 => generated 4 bytes in 10005 msecs (HTTP/1.1 200) 1 headers in 45 bytes (1 switches on core 0)
Accept incoming request: Request_8
[pid: 10|app: 0|req: 1/4] 192.168.65.1 () {32 vars in 405 bytes} [Thu Apr 11 05:04:21 2024] GET /?param=Request_6 => generated 4 bytes in 10008 msecs (HTTP/1.1 200) 1 headers in 45 bytes (1 switches on core 0)
Accept incoming request: Request_2
[pid: 9|app: 0|req: 1/5] 192.168.65.1 () {32 vars in 405 bytes} [Thu Apr 11 05:04:21 2024] GET /?param=Request_1 => generated 4 bytes in 10016 msecs (HTTP/1.1 200) 1 headers in 45 bytes (1 switches on core 0)
Accept incoming request: Request_11
```

(Observe the `Accept incoming request:` lines in the above `uWSGI` server output).

Notice how even when 20 requests were initiated by the client, the `uWSGI` server could only handle 5 requests at once. All the 5 `worker processes` were tied up. Until those requests were not served, no incoming request was being handled. As soon as a worker process became free, it took up another incoming request.


<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

Python's `GIL` limitation is mitigated to some extent by `uWSGI's` ability to spawn multiple worker processes, however it is not enough when dealing with computationally intensive operations.

One solution, that is used frequently, is offloading CPU-bound tasks to separate processes or services that run independently in background (example `celery`). This allows the uWSGI processes to handle incoming requests more efficiently.

The evolution of web frameworks and standards continues to address the limitations observed in traditional `WSGI`-based applications, with `ASGI` being the latest trend.

`ASGI` offers a solution to the synchronous limitations of `WSGI`, enabling Python applications to handle a large number of concurrent connections efficiently. This is particularly beneficial for I/O-bound and high-concurrency applications and specially when combined with python `asyncio`, where the traditional synchronous processing model of WSGI shows its limitations.


<!-- TOC --><a href="#" name="references-and-further-reading"></a>
## References and Further Reading

- [WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface){:target="_blank"}
- [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html){:target="_blank"}
- [ASGI](https://asgi.readthedocs.io/){:target="_blank"}
- [asyncio](https://realpython.com/async-io-python/){:target="_blank"}
- [celery](https://realpython.com/asynchronous-tasks-with-django-and-celery/){:target="_blank"}


