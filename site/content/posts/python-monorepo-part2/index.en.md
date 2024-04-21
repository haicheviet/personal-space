---
title: "Embracing Python Monorepos: Part 2 - Streamlining Development with Docker and GitHub Actions"
subtitle: ""
date: 2023-08-21T14T14:38:51+07:00
lastmod: 2023-08-21T14:38:51+07:00

description: "This article shows how to create python monorepo in industry grade - Part 1."

resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image.webp"

tags: ["Machine Learning", "FastAPI", "Docker", "Feature Store", "Multi-stage", "Project Template"]
categories: ["monorepo-series"]
---


In our previous post, we explored the benefits of using a monorepo for Python projects and set up a basic structure for our repository. In this post, we'll dive deeper into two essential tools that will take our monorepo to the next level: Docker multi-stage builds and Pytest with coverage reporting.

<!--more-->

## Docker Multi-Stage Builds for Python

Docker is an essential tool for any Python developer, allowing us to create isolated environments for our projects. One of the most powerful features of Docker is multi-stage builds, which enable us to create smaller, more efficient images for our applications.

In a Python project, a multi-stage build typically consists of two stages:

- Build stage: This stage is responsible for building our Python application, including installing dependencies and compiling code.
- Runtime stage: This stage creates a minimal image that includes only the necessary dependencies for our application to run.

Here's an example of a Dockerfile that demonstrates a multi-stage build for a Python application, more details for docker multi-stage build you can read [this guide](https://pythonspeed.com/articles/multi-stage-docker-python/):

```docker
# Build stage
FROM python:3.9-slim as build

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

RUN pip install .

# Runtime stage
FROM python:3.9-slim
COPY --from=build /opt/venv /opt/venv

WORKDIR /app

COPY --from=build /app .

CMD ["python", "app.py"]
```

But for our monorepo projects that consist of multiple external dependencies, the above method will not work, for examle fastapi-ml require all libs dependency:

```py
import pydantic
import torch
from ml.core import sum_two_tensor

@app.get("/add-operator")
async def add_operator(a: float, b: float) -> float:
    result: float = sum_two_tensor(torch.tensor(a), torch.tensor(b)).item()
    return result
...
```

To enable caching and build docker efficenly with external project, we require [build-context](https://docs.docker.com/build/building/context/) feature in docker that support using local context to build docker image.

For this monorepo, docker multi-stage can be segment into two steps:

- Step 1: Add your libs to docker build context, ex: `docker buildx build --build-context ml=../../libs/ml ...`
- Step 2: Using `COPY` command to copy local context file into docker build

All the process can be describe as follow:

```bash
# Build the compile stage:
docker buildx build --file Dockerfile \
       --target compile-image \
       --label git-commit=$CI_COMMIT_SHORT_SHA \
       --build-context telemetry=../../libs/telemetry \
       --build-context ml=../../libs/ml \
       --build-context mq=../../libs/mq \
       --cache-from $DOCKER_IMAGE:compile-stage-$TAG \
       --tag $DOCKER_IMAGE:compile-stage-$TAG .


# Build the runtime stage, using cached compile stage:
docker buildx build --file Dockerfile \
       --target runtime-image \
       --label git-commit=$CI_COMMIT_SHORT_SHA \
       --build-context telemetry=../../libs/telemetry \
       --build-context ml=../../libs/ml \
       --build-context mq=../../libs/mq \
       --cache-from $DOCKER_IMAGE:compile-stage-$TAG \
       --cache-from $DOCKER_IMAGE:$TAG \
       --tag $DOCKER_IMAGE:$TAG .
```

```docker
FROM python:3.8.16-slim-buster as base-img
...
# Install dependency
ARG INSTALL_TEST=false
COPY poetry.lock pyproject.toml ./
RUN sed -i 's/develop = true/develop = false/g' pyproject.toml poetry.lock # Change develop to build deployment

COPY --from=telemetry . ../../libs/telemetry
COPY --from=ml . ../../libs/ml
COPY --from=mq . ../../libs/mq
RUN . /opt/venv/bin/activate && poetry install --no-root --only main
```

By using docker con

## Pytest with Coverage Reporting for Multiple Projects



