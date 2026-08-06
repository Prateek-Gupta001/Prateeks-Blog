---
title: "Scaling an LLM Gateway with a semantic cache."
date: 2026-08-06T20:00:00+05:30
draft: false
---

So this is about an LLM Gateway that I built ([AI Gateway](https://github.com/Prateek-Gupta001/AI_Gateway)) — a thin standalone server that sits between you and the llm providers.

It provides a unified API for calling different models. It also has a semantic cache plugin that caches repeating queries across users.

## How the Gateway Works

The basic flow of how the gateway works is:

Request comes in -> check if cacheable -> start embedding generation in the background -> if embed gen takes too much time go to llm response generation -> (in the background if embed gen succeeds in less than 2 seconds do lazy caching) -> check semantic cache -> if cache hit give response to user -> if miss then go to llm response generation and give streaming response to the user.

The project is easy to describe but the engineering challenge was to make the latency overhead of the AI Gateway minimal.

## Performance

Currently the Gateway provides 98x speedups on cache hits.

The embed generation hotpath adds at max 500ms of latency. (if embedding generation takes more than 500ms we go straight to LLM response generation with a cache miss)

Other engineering complexities of the system include async db inserts so that they don't block the hot path.

Lazy caching when embedding generation takes too much time. This requires firing a separate go routine that captures llm response and embedding generation result and checks the cache again .. if doesn't exists ... adds it to the cache.

## The Hardest Technical Roadblock

In this blog I want to talk about the hardest technical roadblock that I faced here:

### Embedding Generation

First I tried using a fastembed library which was basically a Go wrapper that used ONNX runtime library for C and gives you access to those models.

I used the most starred repo for implementing it but upon testing it everything was coming out to be a cache hit. Upon manual introspection by coding up a cosine similarity function myself I found out it's embedding generation results were corrupt (that repo has been archived as of now).

Then I moved to cybertron Go.. which is a pure Go implementation for embedding models .. and it worked well .. it produced good results .. but the resulting architecture was super slow. I then load tested my AI Gateway using a Python script and it was choking on it's responses. Then I profiled the CPU using pprof .. and found out that the matrix multiplications were completely choking my CPU and it wasn't able to handle the requests properly ... I knew what I had to do at that moment.

I decoupled the go monolith to a microservice based architecture and spun up a lightweight gRPC Python embedding microservice. I chose python because it had a more mature ecosystem of embedding libraries and setting them up would guarantee better embed gen results that I could trust.

I used gRPC to minimize latency caused by the decoding and encoding of JSON payloads.

### What This Achieved

This did a few things:

1. This isolates embedding generation and prevents it from crashing the whole system.
2. This allows embedding generation (the most time consuming part of the whole process) to be scaled independently.
3. If the embedding microservice fails then the system can fail openly (the cache can just be turned off).

## Conclusion

This I would say is the hardest technical roadblock I hit while engineering this complicated project, which I engineered my way around by using tools to inspect the bottlenecks of where the problem was coming from and what exactly was causing it, and then choosing the right tech to solve the problem while knowing what needed to be optimized for (latency).