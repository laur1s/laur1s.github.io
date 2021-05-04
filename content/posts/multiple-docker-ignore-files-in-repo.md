---
title: "Multiple Docker Ignore Files in Repo"
date: 2021-04-20T09:45:17+03:00
draft: true
---

## What is a `.dockerignore` file

`.dockerignore` can be used to specify the files and folders that will be excluded from the docker context(and therefore won\'t be added to your container image after using `ADD` or `COPY` commands) For more information about the `.dockerignore` please refer to the [docker docummentation](https://docs.docker.com/engine/reference/builder/#dockerignore-file).


For example for Node.js application my `.dockerignore` file might look something like this:
{{< figure src="/images/dockerignore.png" alt="image" caption="example .dockerignore file for nodejs application" >}}

## What if I have multiple Dockerfiles in my repo and want to ignore different files

