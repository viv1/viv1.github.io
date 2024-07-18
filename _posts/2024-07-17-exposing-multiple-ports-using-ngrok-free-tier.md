---
layout: post
title: Exposing multiple local ports to public with ngrok free tier
date: 2024-07-17 19:24
category: Networking
tags: ["ngrok", "multiple ports", "ssh", "vnc", "expose local network", "raspberry pi", "tunneling", "free"]
description: Ngrok allows only a single agent session at any time on its free tier. In this guide, we'll explore how to expose multiple ports simultaneously.
---


<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Solution](#solution)
- [Steps](#steps)
- [Conclusion](#conclusion)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

I have a multi purpose headless raspberry pi for casual home automation tasks, accessing my hard disks and my development experiments. It runs headless, just connected to a power outlet and my home wifi, with no input or output devices, i.e. no keyboards, no mouse, no monitors. However, since it is connected to my local network, I can easily ssh into it to get access to its terminal. 

I also run a VNC server on it using [RealVNC](https://www.realvnc.com/en/), which helps me view it's UI through the VNC client running on my personal laptops. With this setup, my headless raspberry pi works seamlessly as a fully fledged PC when I am at home.

But I am not at home all the time. There are times when I am away at vacation, or travelling, and I want to access photos on my drive, or continue with my experiments.

<!-- TOC --><a href="#" name="solution"></a>
## Solution

So, how do I expose my port `22` (`ssh port`) and port `5901` (`VNC port`) running on my local network, to the internet so that I can access it from anywhere I want ?

Well, I use [ngrok](https://ngrok.com/) to create secure tunnels to expose my locally running servers to the public internet. There are many similar ones out there, in fact, we can create one for ourselves if we have a publicly available remote instance.

In my case, I use `ngrok` for ease of use, and familiarity with it, since I have been using it for over 6 years, i.e. since the time I was introduced to it during work. We used it to mock a remote server for testing OAuth2 integration flows. 

<!-- TOC --><a href="#" name="steps"></a>
## Steps

1 -  Create a [ngrok](https://ngrok.com/) account, and get your [`auth-token`](https://dashboard.ngrok.com/get-started/your-authtoken)

2 -  Run `ngrok config add-authtoken <TOKEN>` to add your auth token to the ngrok config in your system. 
You can find the config file by running the following:

```bash
$ ngrok config check
Valid configuration file at /home/pi/.config/ngrok/ngrok.yml
```

3 - Run `ngrok tcp 22`. This will create a tunnel on port `22`.
You should see a similar output:

```bash
$ ngrok tcp 22
ngrok                                                                                            

Session Status                online
Account                       Wewake (Plan: Free)
Update                        update available (version 3.12.1, Ctrl-U to update)
Version                       3.12.0
Region                        India (in)
Web Interface                 http://127.0.0.1:4040
Forwarding                    tcp://0.tcp.in.ngrok.io:11317 -> localhost:22

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

4 - You can now SSH to your machine over the internet by:

```bash
ssh pi@0.tcp.in.ngrok.io -p 11317
<Enter password when prompted>
```

5 - Now, to expose the VNC port, if you try running the ngrok command again, you will see an error:

```bash
$ ngrok tcp 5901
ERROR:  authentication failed: Your account is limited to 1 simultaneous ngrok agent sessions.
ERROR:  You can run multiple tunnels on a single agent session using a configuration file.
ERROR:  To learn more, see https://ngrok.com/docs/secure-tunnels/ngrok-agent/reference/config/
ERROR:  You can view your current agent sessions in the dashboard:
ERROR:  https://dashboard.ngrok.com/tunnels/agents
ERROR:
ERROR:  ERR_NGROK_108
ERROR:  https://ngrok.com/docs/errors/err_ngrok_108
ERROR:
```

As can be seen from the error message, ngrok free tier only allows a single agent session. So, how do I expose multiple ports ? Well, you can use teh ngrok config file.

6 - Optionally, add run a simple python script to expose port 8080 as well:

Create file `server1.py`:

```py
from http.server import HTTPServer, SimpleHTTPRequestHandler
server = HTTPServer(('localhost', 8080), SimpleHTTPRequestHandler)
print("Server running on port 8080")
server.serve_forever()
```

and run this server in a separate terminal window:

```bash
python server1.py
```

This creates a server running on port 8080

7 - In the configuration file we saw in step 2, add additional tunnels similar to the following:

```yaml
version: "!"
authtoken: <TOKEN>

tunnels:
  first:
    addr: 22
    proto: tcp
  second:
    addr: 5901
    proto: tcp
  third:
    addr: 8080
    proto: http
```

This add configuration for exposing ports 22 and 5901 over tcp and port 8080 over http.

8 - Run the following command to start ngrok and expose all the configured tunnels:

```bash
ngrok start --all
```

You will see an output similar to this:
```bash
Session Status                online
Account                       Wewake (Plan: Free)
Update                        update available (version 3.12.1, Ctrl-U to update)
Version                       3.12.0
Region                        India (in)
Web Interface                 http://127.0.0.1:4040
Forwarding                    tcp://0.tcp.in.ngrok.io:12541 -> localhost:5901
Forwarding                    tcp://0.tcp.in.ngrok.io:18084 -> localhost:22
Forwarding                    https://05a7-49-47-8-237.ngrok-free.app -> http://localhost:8080

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

You can now access your local servers through these ngrok URLs from anywhere. You can connect to through your VNC client using `tcp://0.tcp.in.ngrok.io:12541`,
ssh using `ssh pi@0.tcp.in.ngrok.io -p 18084` and access your locally running server using `https://05a7-49-47-8-237.ngrok-free.app`.

> Remember to keep your ngrok URLs private, as they provide direct access to your local services.
{:.prompt-info}

9 - Note that you cannot expose more than 3 ports at a time. If you try to add a `fourth` tunnel, you will get:

```bash
ERROR:  failed to start tunnel: Your account may not run more than 3 tunnels over a single ngrok agent session.
ERROR:  The tunnels already running on this session are:
ERROR:  tn_2jNwBYsnoaet8q9ncLvs3O3PNuV, tn_2jNwBb0gy0gw5llswC6xAiVE70a, tn_2jNwBaNCG8HtWR5A7mp1Qdw9bZB
ERROR:
ERROR:
ERROR:  ERR_NGROK_324
ERROR:  https://ngrok.com/docs/errors/err_ngrok_324
ERROR:
```

<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

We can utilize the ngrok configuration file to expose upto 3 ports from locally running machine using the ngrok free tier. This is specially useful for personal projects and tasks, without costing a single penny.