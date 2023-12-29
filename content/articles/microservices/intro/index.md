---
title: "Production Microservices in .NET -- Introduction"
date: 2023-12-26
draft: false
description: "Introduction to the distributed appliacation series with a focus on .NET"
tags: ["microservices", "distributed", ".NET", "dotnet"]
series: ["microservices"]
series_order: 1
image: "img/architect_1.png"
---

## Our Journey Begins

Welcome to this introduction to distributed architecture and microservices in .NET. This is the first post in a series that will serve to chronicle a complete, production-grade deployment of a microservices architecture. But, in addition to reaching a fully deployed application with many features required by a multitude of domains, I also want to highlight the developer experience / headaches to reach that goal since I am, after all, a developer. 

This is mainly a resource to chronicle the decisions I have made, provide some archive to document steps to reproduce, and to share this knowledge with my future self who will likely need it.

## Components

I've worked with many different tech stacks, both front end and back end, among them being NodeJS, Rust, Python, and __.NET.__ I've decided to focus on .NET for this series and for my own distributed applications because I feel like it strikes the right balance between performance, maturity, features, flexibility, etc... I also like that .NET does impose some opinionated patterns on developers which is good for consistency and I believe helps enterprises as their dev teams grow when they are growing beyond the start-up phase.

So here are the main technologies I intend to work with during this series, but it could be subject to change (if it does and I remember, I'll update this list).

### Back end
- [__.NET__](https://dotnet.microsoft.com/en-us/)
- [MassTransit](https://masstransit.io/)
- [RabbitMQ](https://www.rabbitmq.com/)
- [YARP](https://microsoft.github.io/reverse-proxy/)
- [docker](https://www.docker.com/)
- [Kubernetes](https://kubernetes.io/)
- [Helm](https://helm.sh/)
- [Skaffold](https://skaffold.dev/)
- [linkerd](https://linkerd.io/)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)
- [GitHub](https://github.com/)
- I'm still debating on [GraphQL](https://graphql.org/) and [HotChocolate](https://chillicream.com/docs/hotchocolate/v13)
  - I believe I will implement it for this series, but I'll discuss why I question its usage from an architectural / theoretical perspecive
  - However, theory is often great in a vacuum but practical concerns justify breaking some of those theoretical rules

### Front end
Not sure how much this will feature into this series, but for client side:
- [React](https://react.dev/)
- [Flutter](https://flutter.dev/)

### Architecture Overview

This is a very high level overview of what we'll be building:

{{< figure
    src="distributed_overview.webp"
    alt="Distributed Architecture Overview"
    caption="Distroibuted Architecture Overview"
>}}

So it seems simple right?

- K8S Cluster (duh)
  - ingress (duh, we want to communicate with our app)
- YARP
  - Hey, why are you using another reverse proxy after your ingress?
  - Can you just use YARP as your ingress?
    - [Technically Yes](https://github.com/microsoft/reverse-proxy/blob/main/docs/docfx/articles/kubernetes-ingress.md)... maybe we will, or maybe we'll drop YARP. But, I found using a product like [ingress nginx](https://github.com/kubernetes/ingress-nginx) and YARP separately provided me some useful features.
- Rabbit MQ
  - This will likely not end up in our cluster, but in local dev environments we will deploy into our cluster
  - For prod environments, we will use a managed service for resilience.
  - You could cluster this in your app and deploy into your cluster, but it's worth the money to use a managed message bus.
- Identity Microservice
  - I don't care what business you have, there will be identity requirements. We will manage them centrally here.
  - We will have to cluster this microservice as it will be mission critical and could be a single point of failure.
- Users Microservice
  - Again, *almost* every web app has some sort of user profiles (that aren't authentication / authorization related). We will manage those here
- Business microservices
  - This is the rest of your business logic. Regardless of what domain you're in, this architecture can scale to suit your needs.

### Next Steps
With all that out of the way, let's start our journey by choosing the right tools for our development cycle and getting our local dev environment set up.

{{< figure
    src="tools_everywhere.png"
    alt="Choosing the right tools"
>}}

This might sound trivial but we will begin using [Skaffold](https://skaffold.dev/) in the next section so our dev environment will mimic more closely our production deployment.