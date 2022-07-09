---
title: "Mlops Best Practices"
subtitle: ""
date: 2022-07-05T14:38:51+07:00
lastmod: 2022-07-05T14:38:51+07:00
author: ""
description: ""

resources:
- name: "featured-image"
  src: "featured-image.jpg"

tags: ["ECS", "AWS", "Fargate", "Cloudformation", "IAC"]
categories: ["AWS"]
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

After we pick the right format model, we will use Docker to package our code and serve to end user. The tradition Docker build is not dynamic caching and huge in image size that [cost a lot in develoment](https://renovacloud.com/how-to-reduce-your-docker-image-size-for-a-faster-build-deploy/?lang=en). To keep inference size low and enable caching for faster re-build, we will use [multi-stage builds](https://pythonspeed.com/articles/smaller-python-docker-images/)

## Feature Store

## Autoscale app


## Monitoring and aggrage log

- Prediction distribution
- Failure rate
- sdfs


link for reading: https://evidentlyai.com/blog/tutorial-1-model-analytics-in-production