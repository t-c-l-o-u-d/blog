---
layout: post
title: "Second Post?"
date: 2025-01-25 12:37:00 -0600
categories: bash
---

A second post, just for testing.

1. Bash
    ```bash
    $ echo "Hello world!"
    ```
2. Containerfile with no indentation
```Containerfile
RUN echo "Hello world!"
```
3. Dockerfile
    ```Dockerfile
    RUN echo "Hello world!"
    ```
4. docker with FROM and space and no indentation
```docker
FROM example:latest

RUN echo "Hello world!"
```
5. Containerfile with FROM
    ```Containerfile
    FROM example
    RUN echo "Hello world!"
    ```
6. docker with FROM and space
    ```docker
    FROM example
    
    RUN echo "Hello world!"
    ```
7. Python
    ```python
    print(f'Hello world!')
    ```
