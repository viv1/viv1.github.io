---
layout: post
title: "Why Your Python Logs Vanish in Docker (and How PYTHONUNBUFFERED=1 Saves the Day)"
date: 2025-12-26 05:30
category: Python
tags: ["python", "docker", "kubernetes", "logging", "debugging", "containers", "pythonunbuffered", "tty", "terminal"]
description: "Your Python app works fine locally, but in Docker, the logs are MIA. Here's why Python's output buffering drives container users crazy—and how one environment variable fixes everything."
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [The Mystery: Why are my logs disappearing ?](#the-mystery-why-are-my-logs-disappearing-)
- [What is Output Buffering?](#what-is-output-buffering)
   * [Why Buffering Exists ?](#why-buffering-exists-)
   * [The Three Types of Buffering](#the-three-types-of-buffering)
- [Why Docker Makes This Worse](#why-docker-makes-this-worse)
- [The Solution: PYTHONUNBUFFERED=1](#the-solution-pythonunbuffered1)
- [Alternative Solutions](#alternative-solutions)
   * [Use python unbuffered commandline option](#use-python-unbuffered-commandline-option)
   * [Flush Manually](#flush-manually)
   * [Use stderr in Python's logging module](#use-stderr-in-pythons-logging-module)
- [Conclusion](#conclusion)
- [References](#references)

<!-- TOC end -->

## The Mystery: Why are my logs disappearing ?

This has unfortunately happened almost every time I have come back to using python in my little side projects, or experiments.

I create a neat little Python app. It runs perfectly, printing helpful logs at every step:

```python
# app.py
print("Starting application...")
print("Connecting to database...")
print("Processing request...")
print("All done!")
```

I run it locally:

```bash
$ python app.py
Starting application...
Connecting to database...
Processing request...
All done!
```

All good! I containerize it:

```dockerfile
FROM python:3.12-slim
COPY app.py .
CMD ["python", "app.py"]
```

Build and run it in Docker:

```bash
$ docker build -t myapp .
$ docker run myapp
$ docker logs <container-id>
# 
```

But the logs are nowhere to be seen. Where did it go ? If running k8, I check `kubectl logs` , same thing. No logs.

The logs, like a magician's trick, have vanished into thin air. What sorcery is this?

**Spoiler**: It's Python's output buffering, and it's been frustrating developers since Docker became popular.

## What is Output Buffering?

Output buffering is when your program doesn't immediately write output to its destination (like your terminal or Docker logs). Instead, it collects output in a temporary buffer (a chunk of memory) and writes it all at once.

Think of it like this:

**Without buffering**:
```sh
[Your code] ---> "Hello" ---> [Terminal]
            ---> "World" ---> [Terminal]
            ---> "!!!"    ---> [Terminal]
```

**With buffering** :
```sh
[Your code] ---> "Hello" --\
            ---> "World" ---}--[Buffer]---> [Terminal] (when full or flushed)
            ---> "!!!"   --/
```

### Why Buffering Exists ?

Buffering is a performance optimization. Writing to output (terminal, files, network) is **slow** compared to memory operations.


### The Three Types of Buffering

Python (and most languages) uses three buffering modes:

1. **Unbuffered**: Write immediately (no buffer)
   - Used for: stderr by default
   - Behavior: Every `print()` appears instantly

2. **Line-buffered**: Write after each newline (`\n`)
   - Used for: stdout **when connected to a terminal** (TTY)
   - Behavior: Output appears after each line

3. **Block-buffered**: Write when buffer is full (default: 8KB)
   - Used for: stdout **when NOT connected to a terminal** (pipes, files, Docker)
   - Behavior: Output appears in chunks, or when program exits

**Note**: Python detects if stdout is connected to a terminal. If yes, it line-buffers. If no (like in Docker), it block-buffers.

**Want to see the actual buffer size?** You can check it yourself:

```python
import io
print(f"Default buffer size: {io.DEFAULT_BUFFER_SIZE} bytes")
# Output: Default buffer size: 8192 bytes
```

This `io.DEFAULT_BUFFER_SIZE` constant (defined in [Python's io module](https://docs.python.org/3/library/io.html#io.DEFAULT_BUFFER_SIZE)) is what Python uses for block-buffering. It's typically 8192 bytes (8KB), though it can vary based on the system's block size.


## Why Docker Makes This Worse

When you run Python in a Docker container:

1. **stdout is NOT a TTY** (it's a pipe to Docker's logging driver)
2. Python switches to **block-buffering** (8KB buffer by default)
3. The logs sit in memory until:
   - The buffer fills up, OR
   - The program exits, OR
   - You explicitly flush

It's a classic case of "works on my machine" syndrome—because your machine *is* a TTY!

## The Solution: PYTHONUNBUFFERED=1

Set the `PYTHONUNBUFFERED` environment variable to any non-empty value. This forces Python to use unbuffered mode for stdout and stderr.

**In Dockerfile**:

```dockerfile
FROM python:3.12-slim

# Add this line!
ENV PYTHONUNBUFFERED=1

COPY app.py .
CMD ["python", "app.py"]
```

**In docker-compose.yml**:

```yaml
version: '3.8'
services:
  app:
    build: .
    environment:
      - PYTHONUNBUFFERED=1
```

**In Kubernetes deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: PYTHONUNBUFFERED
          value: "1"
```

**At runtime**:

```bash
docker run -e PYTHONUNBUFFERED=1 myapp
```

Now our logs will appear instantly:

```bash
$ docker logs myapp
Starting application...
Connecting to database...
Processing request...
All done!
```

> **Fun fact**: The value doesn't matter. `PYTHONUNBUFFERED=1`, `PYTHONUNBUFFERED=true`, even `PYTHONUNBUFFERED=banana` all work. Python just checks if the variable is set and non-empty.
{:.prompt-info}

## Alternative Solutions

While `PYTHONUNBUFFERED=1` is the easiest fix, here are other approaches:

### Use python unbuffered commandline option

The `-u` flag forces unbuffered mode:

```dockerfile
FROM python:3.12-slim
COPY app.py .
CMD ["python", "-u", "app.py"]  # Note the -u flag
```

This is equivalent to `PYTHONUNBUFFERED=1`, just more explicit.

**Pros**: No environment variable needed  
**Cons**: Have to remember to add `-u` everywhere you run Python

### Flush Manually

Call `flush()` explicitly when you want output to appear:

```python
import sys

print("Critical log message", flush=True)  # Appears immediately

# Or flush everything:
sys.stdout.flush()
```

**Pros**: Fine-grained control over when to flush  
**Cons**: 
- Easy to forget
- Clutters your code
- Doesn't help with third-party libraries that use `print()`

### Use stderr in Python's logging module

`stderr` is unbuffered by default.


```python
import logging
import sys

# Configure logging to stderr (which is unbuffered by default)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    stream=sys.stderr  # stderr is unbuffered!
)

logger = logging.getLogger(__name__)

logger.info("Application starting")
logger.info("Connected to database")
logger.error("Something went wrong!")
```

**Why this works**:
1. `stderr` is **unbuffered by default** (even in Docker)
2. Structured logging is better for production anyway
3. You get timestamps, log levels, and better formatting


## Conclusion

The `PYTHONUNBUFFERED=1` environment variable is a tiny fix that solves a huge headache. It forces Python to stop buffering output, so your logs appear immediately in Docker and Kubernetes.

## References

- [Python io Module Documentation (DEFAULT_BUFFER_SIZE)](https://docs.python.org/3/library/io.html#io.DEFAULT_BUFFER_SIZE){:target="_blank"}
- [Python Documentation: `-u` flag](https://docs.python.org/3/using/cmdline.html#cmdoption-u){:target="_blank"}
- [Python io Module: Text I/O Buffering](https://docs.python.org/3/library/io.html#text-i-o){:target="_blank"}
- [Python Logging HOWTO](https://docs.python.org/3/howto/logging.html){:target="_blank"}
- [Python sys.stdout Buffering](https://docs.python.org/3/library/sys.html#sys.stdout){:target="_blank"}
- [The TTY Demystified](https://www.linusakesson.net/programming/tty/){:target="_blank"}

