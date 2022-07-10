---
title: "Machine-learning inference on Industry-standard"
subtitle: ""
date: 2022-07-05T14:38:51+07:00
lastmod: 2022-07-05T14:38:51+07:00
author: ""
description: ""

resources:
- name: "featured-image"
  src: "featured-image.jpg"

tags: ["Machine Learning", "FastAPI", "ECS", "Feature Store", "IAC"]
categories: ["Machine Learning"]
---

Machine learning inference have evolved tremendously in the past several years. With a wide variety of tools and frameworks out there to simplify deployment and logging. But offently, the inference step is usually bypassed and not shiny as building ML model. However in the ML cycle, training ML model only took 20 perentage of full pipeline ML project. Especially serving AI model in end user is rather simple and only for MVC phase. Therefore how we can server AI model in real world and make AI infernce cycle fast both in implement stage, logging and rebuild. We will dive deep in some best-practices in software development and how it can be implemented in AI respective.

<!--more-->
## Introduction

For the simple AI project, we will use twitter sentiment analyst to demonstrate our project.

![Twitter Sentiment](twitter-sentiment.webp "Twitter Sentiment")

The idea of project is getting metadata from twitter url and analyst sentiment of text. After that we will aggregate sentiment text from indivual user to analyst more sentiment for each user.

## Choosing the right format model

For this project, we will use [transformer roberta model](https://huggingface.co/cardiffnlp/twitter-roberta-base-sentiment) to analyst text sentiment
Firstly, we will find optimize model format to boost inference and standardized our inference code. For pytorch model, we used [torch script](https://pytorch.org/docs/stable/jit.html) to tranform our model to jit format. The translate code is in my [github](), you can reproduce and testing by yourself.

![Pytorch comparison](torch-comparison.webp "Pytorch comparison")

The performance boost is not much but for [GPU model](https://www.educba.com/pytorch-jit/) the runtime of torchscript proves to be better than PyTorch.

The serving code is somewhat simple and already in [ScriptModule](https://pytorch.org/docs/stable/generated/torch.jit.ScriptModule.html#torch.jit.ScriptModule) format that will not change even we change our base model.

**Insert Image here**

For futher optimize model inference time, technique such as quantization or pruning can be applied but require dive deep in investigate model architecture and sepecific for certain architecture. [TVM](https://tvm.apache.org/) framework can be used to auto pruning but required more time to choosing the right compile and GPU tuning. The optimize process is very complicated and deserve its own dedicate blog and I will talk about in another time. For pytorch model, no-braniner way is to convert to script format and gain 5->10% percent performance for free

## Restapi and project template

For serving AI model, the most common protocol is Rest API and we will use [FastAPI](https://fastapi.tiangolo.com/) for our serving framwork. FastAPI was the third most loved web framework in [Stack Overflow 2021 Developer Survey](https://insights.stackoverflow.com/survey/2021/#section-most-loved-dreaded-and-wanted-web-frameworks) and support [OpenAPI](https://github.com/OAI/OpenAPI-Specification) out of the box. Furthermore, the combination of Pydantic and Fastapi is very smooth and strong that I encourage most python developer should use.

{{< admonition info >}}

- Auto-generated docs and simple front end for your API allow you to examine and interact with your endpoints with essentially no added work or tools

- Feature rich and performant

- Tight pairing with Pydantic makes specifying and validating data models easy

{{< / admonition >}}

The project template you can checkout in github repo and coding in project have to be followed by standard style and lint check if member violate coding style. You can find auto format script in [format.sh]() and [lint.sh]() check in github. For more python code style, we use [google python code style](https://google.github.io/styleguide/pyguide.html) for this project and here are some brief configurations guide

```toml
[tool.mypy]
ignore_missing_imports = true

[tool.isort]
profile = "black"

[tool.black]
line-length = 88

[flake8]
max-line-length = 88
exclude = .git,__pycache__,__init__.py,.mypy_cache,.pytest_cache,.eggs,.idea,.pytest_cache,*.pyi,**/.vscode/*,**/site-packages/
ignore = E722,W503,E203
```

## Dockerize

After we pick the right format model, we will use Docker to package our code and serve to end user. The tradition Docker build is not dynamic caching and huge in image size that [cost a lot in develoment](https://renovacloud.com/how-to-reduce-your-docker-image-size-for-a-faster-build-deploy/?lang=en).
It was actually very common to have one Dockerfile to use for development (which contained everything needed to build your application), and a slimmed-down one to use for production, which only contained your application and exactly what was needed to run it. This has been referred to as the [builder pattern](https://refactoring.guru/design-patterns/builder). Maintaining two Dockerfiles is not ideal.
To maintain only on docker file, keep inference size low,  doc and enable caching for faster re-build, we will use [multi-stage builds](https://pythonspeed.com/articles/smaller-python-docker-images/) to dockerize AI service

An AI project's Docker usually is constructed by three steps:

- Download AI model
- Install require package
- Serving AI model

**insert image**

The traditional approach is listing all step in linear and each next step depend in the previous. That is why when we increase version of AI model, we have to rebuild all docker step and can not leverate old install package step even we only change AI model.

Multi-stage build enable us to seperate each step to seperate docker that can be reuse and independend. Making each step can be esiy cached and we only rebuild what we changed. Here is some comparison of tradition vs multi-stage build:

|     |Multi-stage |Traditional|
|:---:|:-----:|:----------:|
|Change AI model|11s|31s|
|Last Image size|||

As you can see, the Multi-stage build save us a lot of time in building image and more lightweight serving.
...

## Feature Store

The natural of AI inference is lots of compute usage and timing, to maintain a healthy app and low latency serving API. We have to use a [feature store db](https://www.tecton.ai/blog/what-is-a-feature-store/) to deliver real-time experience for end-user and saving data.

{{< admonition info >}}

The feature store provides a high throughput batch API for creating point-in-time correct training data and retrieving features for batch predictions, a low latency serving API for retrieving features for online predictions.

{{< / admonition >}}

Redis is most often selected as the foundation for the online feature store, thanks to its ability to deliver ultra-low latency with high throughput at scale.


![Feature Store](feature-store.webp "Feature Store")


Our project will use redis as backend to store data and serve if the prediction for specific text is already made. You can extend our class base Backend

## Autoscale app

When you are building a software application or a service, I’m sure you’ve heard of these big words: scalability, maintainability, and reliability. Espcially in AI project that usually to predict the unknow.

Building an reliable service is substainly hard even for large team. To avoid pitfall in build everything on your own, Cloud vendor is more suitable otpion to deploy and scale our app. For small and medium team size, AWS ECS service and fargate is th best candidate to service our app at optimal, you can find more in [previous blog](https://haicheviet.com/ecs-cost-optimization-with-fargate/) that we're ready discussed about how well of this architecture.

By leverage cloud vendor and serverless platform, we can minmize what can go wrong in AI app add avoid three common types of faults:

- Reliability: Fargate let us focus on building applications without managing servers. Every unknow hardward or software problem that make our app down will be notify and replace. ECS control plan will remove fail node and rebuild new node to our cluster.

- Scalability: AWS Auto Scaling monitors our applications and automatically adjusts capacity to maintain steady, predictable performance at the lowest possible cost and dynamic by traffic.

- Maintainability: By leverate IAC and logs group, manage cluster can not be esier. Every service and configuration is managed in git allow us to review and automation all process.

## Monitoring and aggrage log

- Prediction distribution
- Failure rate
- sdfs

## Some extension

-

link for reading: <https://evidentlyai.com/blog/tutorial-1-model-analytics-in-production>