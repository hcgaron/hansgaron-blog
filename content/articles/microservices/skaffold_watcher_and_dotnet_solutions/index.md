---
title: "Optimizing Skaffold's Local Builds For Dotnet"
date: 2024-01-06
draft: false
description: "Optimizing our dotnet solution to play nicely with Skaffold's dev loop."
tags: ["microservices", "distributed", ".NET", "dotnet", "skaffold"]
series: ["microservices"]
series_order: 7
---

## Motivation

This article stands on its own, but if you've been following this series then you will have noticed something annoying after the [previous article](/articles/microservices/linkerd_service_mesh/).

In that article, we meshed our services with `linkerd`, and to do that we made a script that would create four new local tls certificates every time we run `skaffold dev`. But, if you watch closely you'll notice that skaffold rebuilds everything a ton of times when you run skaffold dev; it's so bad that it makes it hard to shut down the process sometimes!

{{< figure
    src="overheating_1.png"
    alt="Reactor Overheating"
    caption="Skaffold dev overheating your computer"
>}}

This happens because of skaffolds [dev loop](https://skaffold.dev/docs/workflows/dev/#dev-loop) and specifically the [file watcher](https://skaffold.dev/docs/workflows/dev/#file-watcher-and-watch-modes). You see, because of how our `common/skaffold.yaml` is set up, Skaffold thinks that __every file in the solution is a dependency for each project__. So, when __any file changes anywhere__, skaffold will rebuild every project you have running, rededploy, etc... This is especially bad when we create 4 new certifiates and it kicks off the dev loop four times.

So, let's look a little at how Skaffold does this and make some optimizations (and concessions... *sigh*) to our `common/skaffold.yaml` to optimize this. It's not a great solution for dotnet projects, but it's what we have.

{{< alert icon="triangle-exclamation" cardColor="#FFCC22" iconColor="#000000" textColor="#000000" >}}
For those who haven't been following the series, you can follow this article without issue. Just know if I refer to `common/skaffold.yaml`, it's because we are using [skaffold modules](articles/microservices/adding_an_ingress/#reorgainze-skaffold-into-modules). But if you want to apply it to a simple skaffold(ed) project, just know I'm talking about your main `skaffold.yaml` file.
{{< /alert >}}

## Goals

- Fix our `build` section in our `skaffold.yaml` so Skaffold only watches __actual__ dependencies for each project
  - reduce the number of build / deploy loops that happen locally
- Fix our `Dockerfile`(s) in our dotnet projects within our solution to go along with this solution
  - This is great until it's not...
  - Right now there's no problem...
  - If your projects are highly independent and they don't reference one another within the solution, this approach is excellent
  - If you need a shared library from within this solution, then some sort of concession must be made... but we won't get there today

## Fix `common/skaffold.yaml`

### The Problem

The problem is if we make a change in a file in one project, we expect skaffold to only rebuild and deploy that project, but instead it will rebuild and deploy __all__ projects. Go run `skaffold dev` and make a small change in your `Users.Service` and notice how all projects are rebuilt and deployed. We want only the `Users.Service` to rebuild and deploy.

The cause is our __build context__ in the `build` section in `skaffold.yaml`:

```yaml {{hl_lines=[4,8],linenostart=1}
build:
  artifacts:
    - image: hcgaron/identity-service-starter
      context: ../../../
      docker:
        dockerfile: Identity.Service/Dockerfile
    - image: hcgaron/users-service-starter
      context: ../../../
      docker:
        dockerfile: Users.Service/Dockerfile
# ...
```

Notice that we have set the build context to the root of our solution (which we did in a [previous article](/articles/microservices/first_microservices/#fixing-skaffold-yaml)).

The build context tells skaffold what the dependencies of this project will be, so any file in the `context` directory (or any sub-directory) is considered a dependency for __each__ project. The skaffold file watcher watches for changes, and rebuilds any project that is dependent on whichever file changed. We have it set so every project depends on every file in the solution... so every change rebuilds __ALL OUR PROJECTS__.

{{< figure
    src="assembly_line_1.png"
    alt="Assembly Line"
    caption="All your projects building again, and again..."
>}}

Now before you get angry and say *"why did you tell me to do it that way!?"*, remember that we were just updating the build context to be consistent with what skaffold assumes, that the build context will be the root of the project. We also were just following the bootstrapped dotnet convention; the dotnet scaffolding tool (or whatever makes that original dockerfile) creates the dockerfile with the *__assumption that you will have the root of the solution as your build context__*. 

Here's the dockerfile you get out of the box:

```dockerfile {{hl_lines=[8,10],linenostart=1, linenos=table}
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["Identity.Service/Identity.Service.csproj", "Identity.Service/"]
RUN dotnet restore "Identity.Service/Identity.Service.csproj"
COPY . .
WORKDIR "/src/Identity.Service"
RUN dotnet build "Identity.Service.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "Identity.Service.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Identity.Service.dll"]

```

Line __8__ above copies the `.csproj` into the docker container, but notice that it needs to navigate into the `Identity.Service` project first! Then, line __10__ is `COPY . .`, which means it copies __EVERYTHING__ from the solution into the `/src` directory (set as our `WORKDIR` on line 7) of our container.

Think about all that extra garbage we don't need bloating our (build) image! 

{{< figure
    src="dumpster.png"
    alt="Garbage"
>}}

Luckily our final image doesn't contain all that, but this is majorly wasteful and will slow down our build if our solution gets really large. Plus, __having this build context is what causes all our extra skaffold builds / deploys in the dev loop__.

## The Solution

### Fixing `skaffold.yaml`

We will fix our `common/skaffold.yaml` so that the `build.artifacts.context` (ie, the docker build context) points to the __root of each project folder, not the root of the solution__, and make sure the `docker.dockerfile` property points just to `Dockerfile` (which is relative to the `context` we set):

```yaml {{hl_lines=[5,7,9,11,13,15],linenostart=1}
#...
build:
  artifacts:
    - image: hcgaron/identity-service-starter
      context: ../../../Identity.Service
      docker:
        dockerfile: Dockerfile
    - image: hcgaron/users-service-starter
      context: ../../../Users.Service
      docker:
        dockerfile: Dockerfile
    - image: hcgaron/web-bff-starter
      context: ../../../BFF.Web
      docker:
        dockerfile: Dockerfile
#...
```

### Fixing Dockerfile

We need to make sure the `Dockerfile` for each project does two things:
- References files in the correct place, relative to the build context at the project root
- Copies files from the correct directory into the container

Here's the new `Dockerfile` for one project... do the same for all the projects:

```Dockerfile {{hl_lines=[8, "11-12"],linenostart=1, linenos=table}
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["Identity.Service.csproj", "Identity.Service/"]
RUN dotnet restore "Identity.Service/Identity.Service.csproj"

WORKDIR "/src/Identity.Service"
COPY . .

RUN dotnet build "Identity.Service.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "Identity.Service.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Identity.Service.dll"]

```

Notice how on line __8__ we've removed the prefix from the first entry in the `COPY` array, because that file is now at the root of our build context.

For the same reason, we've swapped lines __11__ and __12__. This is because we only want to `COPY` the files from the `Identity.Service` into the container. This not only speeds up our build, but if we *didn't* do this, `docker build` would throw an error, because you can't access files outside the build context (but, later in this series we might do just that...).

Go run `skaffold dev -f ./K8S/skaffold/skaffold.yaml`, then after everything is deployed make a small change in one project... __SUCCESS__, notice that only one project was rebuilt and deployed! No more garbage!

{{< figure
    src="no_more_garbage.png"
    alt="No more garbage"
>}}


{{< alert icon="triangle-exclamation" cardColor="#FFCC22" iconColor="#000000" textColor="#000000" >}}
There is some funkiness with Helm charts being upgraded, for reasons I'm not familiar. I'll have to look into it more, but isn't causing me too much stress yet.
{{< /alert >}}

## Next Steps

One last bit of architecture stuff to take care of before getting into some real coding...

Up next we will enable TSL / HTTPS for our Kubernetes ingress!

After that, I think it's time we started on scalable and secure authentication. Up next we will start creating logic in our `Identity.Service` and our `BFF.Web` project to allow authentication at the ingress. Let's GOOOOOO!

{{< figure
    src="victory.png"
    alt="Victory"
>}}