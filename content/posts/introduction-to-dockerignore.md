---
title: "Introduction to .dockerignore"
date: 2021-04-20T09:45:17+03:00
---

.dockerignore is a special file where you can specify files and folders that will be excluded from the docker context. What it means is that these files won\'t be added to your container image after using `ADD` or `COPY` commands. For more information about the .dockerignore please refer to the [docker documentation](https://docs.docker.com/engine/reference/builder/#dockerignore-file).

For example for Node.js application my .dockerignore file might look something like this:

```Docker
# .dockerignore
# ignore git
.git/
.gitignore
# ignore tests
test/
# local node modules and debug log
node_modules/
npm-debug.log
```

In this case even when I use `COPY . .`  in my Dockefile git, test, node_modules and other directories that I don\'t need in my docker image won\'t be added there resulting in a smaller image sizes.

## Distinct .dockerignore file for each Dockerfile

It\'s quite common to have multiple Dockerfiles in one git repository e.g.: when working with microservices or perhaps you have both frontend and backend code in one repository. In that cases you might want to have separate .dockerignore files for these projects. The question you might have is how docker decides which .dockerignore file to use for each dockerfile. The answer is quite straightforward (However I couldn\'t find this documented anywhere): Docker looks for a .dockerignore that has the same prefix as the dockerfile name.

For example if I have two Dockerfiles: Dockerfile and frontend.Dockerfile:
```sh
├── Dockerfile
└── frontend.Dockerfile
```
I can create a two separate dockerignore files: Dockerfile.dockerignore that Docker will use when building an image from Dockerfile and frontend.Dockerfile.dockerignore that will be used when Docker builds image from frontend.Dockerfile:
```sh
├── Dockerfile
├── Dockerfile.dockerignore
├── frontend.Dockerfile
└── frontend.Dockerfile.dockerignore
```

## Summary

This post briefly explained what is a .dockerignore file and why it\'s  a good idea to use it. I also explained how it works when you have multiple dockerfiles in one directory and want to use specific dockerignore files for each file.
