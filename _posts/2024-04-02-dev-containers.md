---
layout: post
title:  "Dev Containers for setting up Cloud GPU"
date:   2024-04-02 10:32:02 +0200
categories: jekyll update
---
![Dev Containers Logo](/_posts/devc_logo.png)
So you are developing a project that involves training a model. But each time you run it on a cloud GPU, you have to spend +30 min. to prepare the environment.
I show you here how *Dev Containers* solves that problem. 

__What will you get?__
Within 5 min. you get your VS Code connected to your updatable Linux dev environment, having full access to the GPU. 

__How it works?__
Using Dev Containers, you build a docker image, tailored for your project. You load it on Docker Hub (just once).
The image is downloaded and run on your cloud box, and you connect to it to do your work. The changes you make to your code are pushed to/pulled from your GitHub repos. 

__Now a little about Dev Containers__
It works through a VS Code extension. Withing your project folder, you specify what for a docker image is needed, and some other details. It builds the image (once) and runs it and connects to it as a remote. Usually you get your projects folder _mounted_ within the container, such that the changes are mirrored.

__Let's Begin__
1- Make sure Docker is [installed](https://docs.docker.com/engine/install/) is installed. Install the VS Code [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension.
2- Clone your GitHub project and open it with VS Code.
3- Create a `devcontainer` folder, add the files `devcontainer.json` and `Dockerfile`. 
4- `Dockerfile` can be any valid [Dockerfile](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). Use it to setup your environment. Here I explain a typical usage. 
Docker images is built in layers. You can choose an existing image to build on: 
```
FROM python:3.10.14-bookworm
```
This is Debian Bookworm with Python 3.10.14. 
You can choose [Pytorch](https://hub.docker.com/r/pytorch/pytorch), but if you wanna specify the Python version, start from Python ([full list](https://hub.docker.com/_/python)). 

```
WORKDIR /workspaces  
```
It sets the `cwd` for the  following RUN commands. 

Now installing System dependencies: 
```
RUN apt-get update && apt-get install -y \
ffmpeg \
python3-dev \
python3-venv \
&& rm -rf /var/lib/apt/lists/*
```
`python3-dev` and `python3-venv` are needed to build a python virtual environment. 

If you want to build the docker image on you laptop (with no GPUs), Pytorch, by default,  installs a CPU only _wheel_ in the image. To avoid it, we specify that a wheel of PyTorch compatible with CUDA 11.7 (supporting GPU) should be installed: 
```
RUN pip install torch==2.0.0 -f https://download.pytorch.org/whl/cu117/torch_stable.html
```

Cloning other GitHub repos, if needed:
```
RUN git clone https://github.com/someuser/somerepo
```
Create a python [*virtual environment*](https://docs.python.org/3/library/venv.html), activate it, and install the repo there (optionally in [editable](https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/#working-in-development-mode) mode). 
```
RUN python3 -m venv .venv && . .venv/bin/activate && pip install -e /workspaces/boolformer
```

5- Within `devcontainer.json` you can adjust Dev Containers operation. 
```
"build": {
"dockerfile": "Dockerfile",
"context": ".."
},
```
This specifies the Dockerfile used to build the image. `..` sets the build context (where to look for files) to the parent (project) folder. Build is done just once. Later this image is run as a container.

**Note** The rest of the commands may write to the containers writable layer, but NOT to the image. So the 'costly' commands should be in `Dockerfile` to save time.

```
"workspaceMount": "source=${localWorkspaceFolder},target=/workspaces/assessor,type=bind,consistency=delegated",
```
tells to mount your project folder to `/workspace/assessor` in the container. The changes will be mirrored. 

```
"workspaceFolder": "/workspaces/assessor",
```
tells which folder within the container should VS Code open. 

```
"postStartCommand": "cd /workspaces/boolformer && git checkout assessor && git pull origin assessor"
```
`postStartCommand` is run each time the container runs. Here it pulls to git repo that we cloned in the image. This ensures that the container runs with the latest version of repo.

6- Choose `Dev Containers: Reopen in Container` from command palatte. This builds the image and runs it, as you specified. Choosing `Dev Container: Reopen Folder Locally` stops the container.

7- To be able to use the image on Cloud instances with GPU, we should load the the image to Docker hub. 
Create a (free) Docker hub account. Get an *Access Token* from Account Settings > Security, use it to login: 
```bash
docker login
```
get the image ID:
```
$ docker images 
REPOSITORY TAG IMAGE ID CREATED SIZE 
vsc-... latest 813dc... About an hour ago 15.6GB
```
Use it to tag the image:
```bash
docker tag 813dc339be johndoe/dev-container:v1
```
replacing `johndoe` with you Docker Hub username, and `dev-container` and `v1` with a repo name and tag you like.
Now we can push it to Docker Hub:
```
docker push johndoe/dev-container:v1
```
Check it on Docker Hub.

8- Suppose you are remotely connected to an online GPU with VS Code. Clone your project form GitHub, and open it. Change `devcontainers.json` to use the image:
```
//"build": {
//"dockerfile": "Dockerfile",
//"context": ".."
//},
"image": "johndoe/dev-container:v1",
```
Reopen the project in container, your environment is ready! Do your development and commit/push the changes you made to the git repos. That's it!

**WAIT. What about GPU?**
Oh, Sure :) [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) should be installed on your cloud instance (it usually is). Then just add this to `devcontainers.json`:
```
"runArgs": [
"--gpus=all"
],
```
Remember to comment it out if you run it on a CPU-only instance.

Running this inside the container
```bash
$python -c 'import torch; print(torch.cuda.is_available())'
```
should return `True` if GPU is available.
