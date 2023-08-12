---
title: "Python monorepo - Part 1"
subtitle: ""
date: 2023-08-10T14:38:51+07:00
lastmod: 2023-08-10T14:38:51+07:00

description: "This article shows how to create ML service on Industry-standard."

resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image.webp"

tags: ["Machine Learning", "FastAPI", "Docker", "Feature Store", "Multi-stage", "Project Template"]
categories: ["Machine Learning"]
---


One of the biggest problems in my throughout year of working in tech industry is communication and ensure that everyone on team is onboard with each other. The thing here is not only communication in your team but as your whole ogrianziation as well. Mismatch dependency, API contract violate and intergation test is the most common bugs that I found in production and even countless post-mortern notes and meetings, no one can ensure that the this problem will not occur again. But I think monorepo can be a solution for that, with the potential can lead to better communication and a more cohesive team.

<!--more-->
## Introduction

Have you ever made some minor change in your code, but require to create 4 pull requests across multiple repos and reviews, half a day to test and a couple hours to release. Despite all the nonsense that you have been through, but some motherf\*ckers in other repo think your code is not needed and delete without your consent, create an outage in production just for some minor change in API schemas!!!

![Alt text](image-3.png)

Yup, been there and f\*ck that. This example is not rare and using monorepo can fix this.

The philosophy of monorepo is really simple, keeping all of the code for a project in a single repository. Here is a general principles of monorepo

{{< admonition >}}

`Scopes`: The folders act as scopes to make sure code artifacts are only visible when they should be. This allows to extract common tasks (e.g. building a C# solution) quickly and maintainers can easier reason about where the error lies.

`The One Version Rule (Atomic Commits)`: The principle guarantee that you can commit atomically to both of related projects simultaneously. There is no view of the repository where Project A is at Commit #1 but Project B is at Commit #2.

`Big pictures`: With everything in one place there is no need to copy code between repositories or to look for infrastructure as code files and documentation.

`Good practice`: A monorepo requires teams to work with each other. By merging code only with a MR, teams review each other’s code which breaks silos and improves code quality.

{{< /admonition >}}

![Alt text](image.png "[ref]](https://www.raftt.io/post/development-challenges-of-working-with-monorepos-and-multirepos)")

> TLDR: Monorepo is not a new tech but I find some tooling around python is really poor and rather then a hack than a standard. My blog and [github repo](https://github.com/haicheviet/python-monorepo/blob/main/libs/ml/pyproject.toml) plan to solve that:

* Poetry playwell with docker container.
* Automated formatting and linting for the Python project.
* Docker multi-stage CI/CD pipeline that allows sharing and cache stages between libraries.
* Test and coverage page for whole monorepo including all subprojects.

## Problem Statement

So what the problems monorepo want to solve:

* Improved collaboration: Makes it easier for teams to collaborate on code. Everyone can see all of the code in the project, which makes it easier to find and understand what is going on.
* Reduced complexity: Reduce the complexity of managing dependencies. When all of the code is in a single repository, it is easier to keep track of which dependencies are used by each component of the project.
* Increased productivity: Increase productivity by reducing the time it takes to build and test code. When all of the code is in a single repository, it can be built and tested as a single unit.
* Central CI-CD: Can be used to create a single test suite that tests all of the code in the project. This can help to ensure that the entire project is always working together correctly.

But all the code in one place create another challanges to solve from ops side:

* How can we debug, standardlize flow and make cross change effiencely between multiple projects?
* Managing dependencies in a monorepo can be challenging, as it is important to ensure that all of the dependencies are compatible with each other. This can be especially difficult when the monorepo contains a large number of dependencies that depend each other.
* Each service have diffent way to CI-CD and testing, how can we merge all the pipeline to one flow?
* Monorepo promise that all test case can be tested in one place, that sounds good but painful to implement. For ex: I just change some minor code in my project but every push to MR require full re-run test suites of all projects. Seem inconvenient and too much resource wasted.

The part one of [this series]() will discuss these items and hopefully solve some of the challanges above:

* Project structure
* Project standard (lint,test,packaging)
* Python enviroments management and debug

## Project structure

```markdown
├── .github/workflows           # Github action CI-CD
├── scripts/                    # General scripts
├── libs/                       # Share libs
│   └── ml/
│       ├── ml/                 # Source code
│       ├── tests/              # Unittest cases
│       ├── scripts/            # Build and test scripts
│       ├── pyproject.toml      # Requirement dependency and python metadata
│       └── poetry.lock         # Lock dependency
├── services/
│   └── fastapi-ml/
│       ├── app/                # Application code, mostly about client facing
│       ├── scripts/            # Build, test and deploy scripts
│       ├── tests/              # Unit and integration test cases
│       ├── pyproject.toml      # Requirement dependency and python metadata
│       └── poetry.lock         # Lock dependency
```

One of the most important things to consider when setting up a monorepo is the structure of the top-level folders. These folders should be clearly named and simple, without any special naming conventions. This will make them easier to understand and navigate, even as the project matures over years and thousands of developer hours.

## Project standard

### Typechecking and Linting

I already talked about project code style before in [this blog](https://haicheviet.com/machine-learning-inference-on-industry-standard/#project-code-style). But currently I discover a new tool [ruff](https://github.com/astral-sh/ruff) that solve the problem with flake8. The new config will was shorten and all package in one `pyproject.toml` file.

```toml
[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.black]
line-length = 88

[tool.ruff]
select = [
    "E",  # pycodestyle errors
    "W",  # pycodestyle warnings
    "F",  # pyflakes
    "I",  # isort
    "C",  # flake8-comprehensions
    "B",  # flake8-bugbear
]
ignore = [
    "E501",  # line too long, handled by black
    "B008",  # do not perform function calls in argument defaults
    "C901",  # too complex
]
# Exclude a variety of commonly ignored directories.
exclude = [
    ".bzr",
    ".direnv",
    ".eggs",
    ".git",
    ".git-rewrite",
    ".mypy_cache",
    ".pytest_cache",
    ".ruff_cache",
    ".venv",
    "__pypackages__",
    "__pycache__",
    "build",
    "dist",
    "venv",
]
```

### Testing

Every project will have folder `tests` that contain all the test case such as unit and intergration tests. The tooling for python to test is more well-know and define, pytest and pytest-cov. Pytest is for testing and pytest-cov to generate coverage report

```toml
[tool.poetry.group.test.dependencies]
pytest = "6.2.2"
pytest-cov = "4.0.0"

[tool.pytest.ini_options]
log_cli = true
log_cli_level = "DEBUG"
addopts = "--cov --cov-report term"
testpaths = ["tests"]
```

## Python enviroments management and debug

Python management is really [hell dependency](https://medium.com/knerd/the-nine-circles-of-python-dependency-hell-481d53e3e025#:~:text=%E2%80%9CDependency%20hell%E2%80%9D%20is%20a%20term,are%20shared%20across%20a%20project.), using bare pip package to manage large projects is beyoung my mind. I was skeptical with using [poetry](https://github.com/python-poetry/poetry) in python management but the push toward monorepo in poetry is really strong and most of my complain is already sovled.

First we will talk about how to use poetry as fully to manage our monorepo

### Dependency groups

We should seperate clearly between multiple dependency in our project. For ex: dev, lint, test and main dependency. With [poetry grouping](https://python-poetry.org/docs/master/managing-dependencies/), we can easily achive that:

```toml
[tool.poetry.dependencies] # Main dependency
python = "^3.8.1"
torch = {version = "2.0.1", source="torchcpu"}
telemetry = {path = "../telemetry"}

[tool.poetry.dev-dependencies]
telemetry = {path = "../telemetry", develop = true} # for developing environment


# Test dependency as optional to keep lightweight package
[tool.poetry.group.test]
optional = true

[tool.poetry.group.test.dependencies]
pytest = "6.2.2"
pytest-cov = "4.0.0"

# Lint dependency as optional to keep lightweight package
[tool.poetry.group.lint]
optional = true


[tool.poetry.group.lint.dependencies]
mypy = "1.4.1"
black = "23.3.0"
ruff = "0.0.278"
```

With this poetry config, we can easily switch dependency and minimal installalbe as possible. The flow to manage python environment can be showed below:

{{< mermaid >}}

graph LR
    A[pyproject.toml]
    A -->|Production|   B[poetry install]
    A -->|Development|  C[poetry install --with dev,lint,test]
    A -->|Lint check|   D["poetry install --no-root --only lint"]
    A -->|Test check|   E["poetry install --with test"]

{{< /mermaid >}}

### One python environment rule all

The second interesting choice we made was to use editable installs for libraries. Using poetry, this is done with `path` dependencies: `poetry add ../../lib/grpc`. When we install an internal library this way, everything generally works the same as if we installed it from a package repository — we can import from it, and its dependencies get installed — but it also links back to the local library directory. If you edit the library locally, there’s now no need to reinstall it, solving the problem of library changes being difficult

Using poetry, we can debug and make cross change so smoothly in this demo below


TODO: add gif here



## Some afterthought

In this blog post, we have discussed the structure and standard of a Python monorepo. We have seen how a monorepo can be used to improve collaboration, code sharing, and dependency management. We have also seen how to choose the right tooling and structure for your project.

In the next part of [this serries](), we will discuss the challenges of build docker multi-stage for monorepo. Enable caching and best practice to deploy service.