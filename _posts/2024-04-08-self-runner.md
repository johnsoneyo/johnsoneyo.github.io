---
layout: post
title: Dockerised Self Runner  
---

A self hosted dockerised version of github actions self runner

## Self Runner 

Self runner is a basic agent using ubuntu 18 arch to run github action build events, The self runner hosted is inspired by [Michael Herman](https://github.com/mjhea0)
and modified to work for repositories and not organisations.

### Docker ENV Variables 
- GIT_REPO - Repository to be monitored in github actions
- ACCESS_TOKEN - Access token for the repository  

### Usage 

#### Build image with image tag i.e runner-image

```shell

> docker build --tag runner-image --no-cache .  

```

#### Run instance from image

```shell
> docker run --detach --env GIT_REPO=https://github.com/johnsoneyo/jmodelmapper  --env ACCESS_TOKEN=AB4QZ6WHUBJNHL47HCBMULTGCOWGO --name runner runner-image
```

#### Source 
[https://github.com/johnsoneyo/self-runner](https://github.com/johnsoneyo/self-runner)

### References 
- [https://testdriven.io/blog/github-actions-docker/](https://testdriven.io/blog/github-actions-docker/) 
