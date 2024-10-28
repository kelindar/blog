---
title: Building a Faster, Leaner Vector Search Library in Go
date: 2024-10-23 23:00:00
author: "Roman Atachiants"
categories: [Engineering]
tags:
  [
    search-engine,
    ai,
    gpu,
    embeddings,
    simd,
    semantic-search,
    bert,
    vector-search,
    llamacpp,
    gguf,
  ]
comments: true
cover:
  image: img/cover.png
excerpt: "A story of how I spent a few weekends (and way too much coffee) building a faster, leaner vector search library in Go. After hitting walls with fancy algorithms, I went back to basics, optimized brute-force, squeezed embeddings into ~2ms, and pulled off 8-nanosecond SIMD moves — proving once again that stubbornness is a feature, not a bug."
---

It all started innocently enough with a gaming side project. I wanted to create a fast and clever chat-bot for an NPC (Non-Player Character) that could keep up with players' questions in real-time. Imagine a smart in-game assistant that could respond naturally to player queries without breaking immersion. But, as these things go, the project quickly evolved into something more ambitious. Over weekends and late nights, my simple chat-bot idea evolved into a full-blown vector search library [kelindar/search](https://github.com/kelindar/search).

To make the NPC interaction feel seamless, I needed fast, low-latency search, and I needed it to run right in-process. Offloading the search to external services wasn’t an option—introducing network latency would have killed the experience. The goal was clear: make it fast, make it lean, and keep it all in-process. I began experimenting with what seemed like the right algorithms for scalable semantic search: HNSW and LSH.

![Demo](img/demo.gif)

### Embedding without bloat

For generating embeddings, I played with different models and settled on the [MiniLM v6](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2), Q8 version (quantized) of BERT-based embeddings, each with 384 dimensions. Each embedding operation took under `~2 milliseconds`, making it suitable for real-time search. I've been told at work by my fellow data scientists that 2ms is quite good, but I checked other models as well to confirm. Few of the libraries I checked were significantly slower, even with GPU support so I decided to stick with MiniLM, it was fast and accurate enough for my needs.

To avoid the complexities of `cgo`, I leveraged [purego](https://github.com/ebitengine/purego) library to call shared C libraries directly, simplifying cross-compilation and deployment. To get this out, I also staticly linked the `llama.cpp` library, which is a C++ library for LLMs, but also support embeddings as of last year. This allowed me to generate embeddings without the overhead of `cgo`, keeping the process fast and efficient.

### The search for the right algorithm

Once I had the embeddings, I needed a way to search them efficiently. I explored Hierarchical Navigable Small World (HNSW) and Locality-Sensitive Hashing (LSH). Both are designed to scale efficiently with large datasets and offer sub-linear time complexity. However, while the latency of these methods was more than acceptable, the recall was where things fell short. The probabilistic nature of LSH and the hierarchical layering in HNSW led to inconsistencies in retrieving the most relevant results.

Since the chat-bot needed to consistently provide meaningful responses, high recall was a non-negotiable requirement. So, I pivoted to a simpler but highly optimized brute-force approach, which gave me full control over the search accuracy.

It might sound counterintuitive, but with careful optimizations, brute-force wasn’t the naive choice. Using one of my other libraries, [kelindar/gocc](https://github.com/kelindar/gocc), I generated highly optimized Go assembly from C code and incorporated SIMD operations. The result? Comparisons were performed in just ~30 nanoseconds per pair of embeddings, scaling linearly with dataset size.

### Speeding up similarity

One bottleneck that emerged (in my head) was calculating cosine distance. While 30 nanoseconds seems good enough, there was more CPU cycles left on the table. To further optimize, I pre-normalized the vectors, turning cosine distance calculations into simple dot products. This reduced computation time by roughly 3x, significantly boosting overall responsiveness. As per my last benchmark a SIMD optimized dot product took around `8 nanoseconds` on my machine, using AVX-2 instruction set.

### Final thoughts

The journey of building this library taught me that sometimes simpler solutions, when well-optimized, can outperform more complex algorithms. Returning to a brute-force approach with targeted optimizations met the specific needs of my chat-bot while ensuring high recall and low latency. While this library isn’t designed for massive datasets or advanced query operations, it excels at providing accurate and fast semantic search within its intended scope.

Next steps? I still think there's room for improvement in the basic brute force search by quantizing the embeddings and perhaps using a more sophisticated indexing structure. But for now, I’m happy with the results.
