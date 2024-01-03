---
title: "Fixing Skaffold Modules With A Deployer Definition"
date: 2024-01-02
draft: false
description: "Our Skaffold Modules were broken because our rawYaml wasn't being deployed. Here we fix that easily before moving to our service mesh implementation."
tags: ["microservices", "distributed", ".NET", "dotnet", "linkerd", "service-mesh", "Skaffold"]
series: ["microservices"]
series_order: 5
---

## Interlude -- Fixing `skaffold.yaml`

Remember that __Skaffold Bug Alert__ from [the previous article](/articles/microservices/adding_an_ingress#skaffold-bug-alert)? Let's address it now by __fixing our Skaffold modules once and for all. All we needed to do originally was add `deploy.kubectl: {}` to our `common/skaffold.yaml`. But, because we developed some more, we need to make a few more changes.

- Add a `deploy.kubectl` section to our `common/skaffold.yaml` file
- Uncomment our `requires` section in root level `skaffold.yaml`
- Remove the `build` and `deploy` sections from our root level `skaffold.yaml`
- move the `webbff` stuff of our deploy and build sections into `common/skaffold.yaml` and update their paths, etc...

Instead of making you do that, just copy the contents of these files and replace what you have (or you can do it as an exercise if you feel you must be punished):

`skaffold.yaml`
```yaml
apiVersion: skaffold/v4beta8
kind: Config
metadata:
  name: main-config
requires:
  - path: ./local/skaffold.yaml
    configs: [local-config]
    activeProfiles:
      - name: local
        activatedBy:
          - local
  - path: ./dev/skaffold.yaml
    configs: [dev-config]
    activeProfiles:
      - name: dev
        activatedBy:
          - dev
  - path: ./stage/skaffold.yaml
    configs: [stage-config]
    activeProfiles:
      - name: stage
        activatedBy:
          - stage
  - path: ./prod/skaffold.yaml
    configs: [ prod-config ]
    activeProfiles:
      - name: prod
        activatedBy:
          - prod   
profiles:
  - name: local
    activation:
      - command: dev
      - env: ENV=local
  - name: dev
    requiresAllActivations: true
    activation:
      - command: dev
      - env: ENV=dev
  - name: stage
    activation:
      - env: ENV=stage
  - name: prod
    activation:
      - env: ENV=prod
```

`common/skaffold.yaml`
```yaml
apiVersion: skaffold/v4beta8
kind: Config
metadata:
  name: common-config
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
    - image: hcgaron/web-bff-starter
      context: ../../../
      docker:
        dockerfile: BFF.Web/Dockerfile
manifests:
  rawYaml:
    - ../../manifests/*
deploy:
  kubectl: {}
  helm:
    releases:
      - name: nginx-ingress
        namespace: app
        createNamespace: true
        remoteChart: oci://ghcr.io/nginxinc/charts/nginx-ingress
        version: 1.1.0
        valuesFiles:
          - ../../Helm/nginx-ingress/values.yaml
      - name: identity
        namespace: app
        createNamespace: true
        setValueTemplates:
          image.repository: "{{.IMAGE_REPO_hcgaron_identity_service_starter}}"
          image.tag: "{{.IMAGE_TAG_hcgaron_identity_service_starter}}@{{.IMAGE_DIGEST_hcgaron_identity_service_starter}}"
        setValues:
          image.pullPolicy: "IfNotPresent"
        chartPath: ../../Helm/Identity
        valuesFiles:
          - ../../Helm/Identity/values.yaml
        version: 0.1.0
      - name: users
        namespace: app
        createNamespace: true
        setValueTemplates:
          image.repository: "{{.IMAGE_REPO_hcgaron_users_service_starter}}"
          image.tag: "{{.IMAGE_TAG_hcgaron_users_service_starter}}@{{.IMAGE_DIGEST_hcgaron_users_service_starter}}"
        setValues:
          image.pullPolicy: "IfNotPresent"
        chartPath: ../../Helm/Users
        valuesFiles:
          - ../../Helm/Users/values.yaml
        version: 0.1.0
      - name: webbff
        namespace: app
        createNamespace: true
        setValueTemplates:
          image.repository: "{{.IMAGE_REPO_hcgaron_web_bff_starter}}"
          image.tag: "{{.IMAGE_TAG_hcgaron_web_bff_starter}}@{{.IMAGE_DIGEST_hcgaron_web_bff_starter}}"
        setValues:
          image.pullPolicy: "IfNotPresent"
        chartPath: ../../Helm/WebBFF
        valuesFiles:
          - ../../Helm/WebBFF/values.yaml
        version: 0.1.0
```

Now if you run `skaffold dev -f ./K8S/skaffold/skaffold.yaml`, it should not only start up but it should also deploy our `Ingress` object to do our BFF routing.

Verify by running `kubectl get ingress -n app`:

![ingress](image.png)

And you can still make GET requests to `http://web.hg-microservices-blog.com/api/users` and `http://web.hg-microservices-blog.com/api/auth` without issue.

Great - we have fixed our modules. (A little explanation on what we actually did below)

{{< figure
    src="user_error.png"
    alt="User Error"
>}}

## Skaffold Deployers

Skaffold deployers are responsible for deploying the final Kubernetes manifests to the cluster. Skaffold supports multiple tools for deploying applications, and the configuration for these deployers is set through the deploy section of the skaffold.yaml file. 

When you use a single `skaffold.yaml` file, technically you don't need to define a `deploy` section at all (as long as you are using `rawYaml`). Skaffold is smart enough to pick a default deployer for you (in my experience it's always been `kubectl`, but I don't know if that's always the case or if there's more logic to it). But when using Skaffold modules, *no default deployer is chosen for you*. So, even though we have defined `rawYaml`, we need to define the deployer in the `common/skaffold.yaml` to deploy those `rawYaml`. That's what we have done here.

## Back To The Action

Next we will finally get back to deploying linkerd and inject our sidecar proxies!