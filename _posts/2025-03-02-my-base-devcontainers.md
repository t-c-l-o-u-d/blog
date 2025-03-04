---
layout: post
title: Development Container Base Images
date: 2025-03-02
categories: devcontainers
---

Today I want to introduce one of my [repositories](https://github.com/t-c-l-o-u-d/devcontainer-base-images) regarding Development Containers. Without further ado, let's dive in.

### What Are Development Containers?
TL;DR - They are a portable and reproducible development environment. 

Quoting the project's homepage:

*"A development container (or dev container for short) allows you to use a container as a full-featured development environment. It can be used to run an application, to separate tools, libraries, or runtimes needed for working with a codebase, and to aid in continuous integration and testing. Dev containers can be run locally or remotely, in a private or public cloud, in a variety of supporting tools and editors." - https://containers.dev/*

Containers are great for deploying workloads, but this project took things to a new level by letting you *develop* inside a container.

### The Repository
The project includes numerous default images, but it doesn't cover my exact use case. I wanted a very specific set of items installed into each image and, for that, I needed to create my own.

The repository consists of three images at the time of writing:
1. [Arch Linux](https://ghcr.io/t-c-l-o-u-d/devcontainer-base-images/arch-linux-devcontainer:latest)
	- `ghcr.io/t-c-l-o-u-d/devcontainer-base-images/arch-linux-devcontainer:latest`
3. [Fedora](https://ghcr.io/t-c-l-o-u-d/devcontainer-base-images/fedora-devcontainer:latest)
	- `ghcr.io/t-c-l-o-u-d/devcontainer-base-images/fedora-devcontainer:latest`
5. [Ubuntu LTS](https://ghcr.io/t-c-l-o-u-d/devcontainer-base-images/ubuntu-lts-devcontainer:latest)
	- `ghcr.io/t-c-l-o-u-d/devcontainer-base-images/ubuntu-lts-devcontainer:latest`

These images contain a common set of tools that I would need across nearly any repository. Right now, you won't find things like `python` included because that isn't applicable across all of my repositories. Things aren't perfect, but they are continously maintained through a daily GitHub Actions build. Every part of the process is completely transparent and automated.

Current list of packages:
| Arch Linux      | Fedora                | Ubuntu LTS        |
| --------------- | --------------------- | ------------------|
| 7zip            | 7z                    | 7zip              |
| bash-completion | bash-completion       | bash-completion   |
| git             | file                  | curl              |
| htop            | git                   | file              |
| man-db          | gzip                  | fonts-ibm-plex    |
| man-pages       | htop                  | git               |
| openssh         | ibm-plex-mono-fonts   | gnu-which         |
| progress        | man-db                | htop              |
| rsync           | rootfiles             | man-db            |
| ttf-ibm-plex    | rsync                 | manpages          |
| unrar           | sudo                  | rsync             |
| unzip           | tar                   | sudo              |
| xdg-utils       | unrar                 | unrar-free        |
| .               | unzip                 | unzip             |
| .               | watch                 | xdg-utils         |
| .               | which                 | .                 |
| .               | xdg-utils             | .                 |

Feature parity between the images is a goal, but I can't ensure it right now. I have recently transitioned from Fedora to Arch Linux so the Arch Linux image is the most *battle-tested*.

### Usage
All of these images are a starting point for my own development projects. Taking the `python` example from above, let's take a look at how I integrated it into my [playground-python](https://github.com/t-c-l-o-u-d/playground-python).

After copying `Containerfile.template` and `devcontainer.json.template` into my repository I proceed to edit them.

Starting with my `Containerfile`, I found the clearly labeled section to add my repository's needs. As you can see here, it is just a long list of packages to install.
```docker
...
# =======================================
# repository specific commands start here
# =======================================

# install python tools
RUN pacman --sync --noconfirm \
	autopep8 \
	flake8 \
	mpdecimal \
	mypy \
	python \
	python-astroid \
	python-autocommand \
	python-black \
	python-click \
	python-colorama \
	python-dill \
	python-entrypoints \
	python-flake8-black \
	python-flake8-isort \
	python-importlib-metadata \
	python-isort \
	python-jaraco.collections \
	python-jaraco.context \
	python-jaraco.functools \
	python-jaraco.text \
	python-mccabe \
	python-more-itertools \
	python-mypy_extensions \
	python-orjson \
	python-packaging \
	python-pathspec \
	python-platformdirs \
	python-pycodestyle \
	python-pyflakes \
	python-pylint \
	python-setuptools \
	python-tomli \
	python-tomlkit \
	python-typing_extensions \
	python-wheel \
	python-zipp \
	ruff \
	yapf

# =====================================
# repository specific commands end here
# =====================================
...
```

The `devcontainer.json` file is equally boring and I like that. Boring and simple are great for helping people unfamiliar with your project to quickly grasp how things come together and how they can effectively contribute. The notable thing here is that extensions and settings for VSCode are all contained with the container! This ensures that anyone else who clones this repository and uses the dev container ends up with an exact replica of my environment. No more *"It works on my machine."*
```jsonc
...
"settings": {
	...
	// repository specific settings go here
	"python.analysis.reportExtraTelemetry": false
},
"extensions": [
	"ms-azuretools.vscode-docker",
	// repository specific extensions go here
	"ms-python.autopep8",
	"ms-python.flake8",
	"ms-python.isort",
	"ms-python.mypy-type-checker",
	"ms-python.pylint",
	"ms-python.python"
]
...
```

### Closing Thoughts
Thank you very much for reading, this has been the culmination of a lot of work on my part. Not everyone will agree with my usage of dev containers in this way, but it works for me and hopefully they will work for you!
