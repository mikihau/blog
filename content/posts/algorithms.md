---
title: "???"
date: 2019-03-13T22:08:04-08:00
draft: true
---

more reference: http://web.stanford.edu/class/cs166/handouts/100%20Suggested%20Final%20Project%20Topics.pdf

#### Reservoir Sampling (Randomized Algorithms)
- Problem
    Given a stream of elements too large to fit into memory, pick k random elements from the stream with uniform probability.
- Algorithm
    1. Create a `reservoir` of size k, fill it up with the first k elements of the stream.
    2. For each element (indexed i) thereafter, generate a random integer j from 0 to i (inclusive on both sides). If `0 <= j <= k-1`, replace `reservoir[j]` with this new element.
- Why does it work
    - Say there's n elements in the stream. We want to make sure each element gets chosen with probability `n/k`.
    - For one of the first k items, the probability that it remains until the end of the procedure is `k/(k+1) * (k+1)/(k+2) * ... * (n-1)/n = k/n`.
    - For one of the later items (say it's indexed at i), the probability it gets onto the reservoir is `k/(i+1)`, and the probability that it survives till the end is `(i+1)/(i+2) * (i+2)/(i+3) * ... * (n-1)/n`. Multiply them together we also get `k/n`.
- Implementation
```python
import random

def generate_stream(n=1000):
    return (i+1 for i in range(n))

def reservoir_sampling(k=10):
    stream = generate_stream()
    reservoir = [next(stream) for _ in range(k)]
    i = k
    for element in stream:
        j = random.randint(0, i)
        if 0 <= j <= k-1:
            reservoir[j] = element
        i += 1
    return reservoir

print(reservoir_sampling())
```
- Real world example/where it's used: ???

#### Segment Trees

#### Self-balancing Binary Search Trees
