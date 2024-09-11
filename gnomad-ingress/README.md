# gnomAD ingress configuration

This set of kustomizations configures the ingress object for the gnomAD browser. Because the ingress sits in front of several different services and changes with a much different frequency than our app deployments, we manage it as a separate deployment.

This directory contains a definition for the prod ingress, and a demo base to use for putting an ingress in front front of your demo browser deployments.

## Creating a new demo ingress

Create a new directory at the same level as `demo/`, cd into it, and then run a `kustomize init`:

```bash
mkdir new-feature-ingress && cd new-feature-ingress
kustomize init --resources ../demo
```

In the kustomization.yaml file, you should:
- Add a nameSuffix or namePrefix to identify your ingress
- supply a patch to set an appropriate domain name
- the name of your browser Service object
- add any specific routes that your ingress needs to serve.
  - For example, the default demo kustomization serves all traffic to a single service. You may want to specify a `/reads` route if you need to serve reads data from your demo. The [Common Patches](#common-patches) section has details on how to add this functionality.

## Creating or updating the prod ingress

At the moment, we only have one prod ingress. You can view its configuration by running `kustomize build prod`. The deployment is automated via ArgoCD. If you need to make a change to the prod ingress, create a Pull Request with your changes, and coordinate a deployment through Argo once it's reviewed ang merged.

## Common Patches

Here's some examples of patches that could be placed in your kustomization.yaml file. Each `-op:` element here could be used individually, or you can combine multiple operations into a single `-patch:` objects. A complete example that changes the hostname, adds reads URL paths, and sets custom service names can be found in the `example-demo-reads/` folder.

### Setting a domain name for a demo:

```yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: new-feature.gnomad.the-tgg.dev
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: demo-ingress
```

### Setting a service name to point at a specific service

```yaml
  - patch: |-
      - op: replace
        path: /spec/rules/0/http/paths/0/backend/service/name
        value: demo-browser-newfeature
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: demo-ingress
```

### Adding reads paths to your demo ingress:

```yaml
  - patch: |-
      - op: add
        path: /spec/rules/0/http/paths/-
        value:
          path: "/reads"
          backend:
            service:
              name: demo-reads-service
              port:
                number: 80
          pathType: ImplementationSpecific
      - op: add
        path: /spec/rules/0/http/paths/-
        value:
          path: "/reads/*"
          backend:
            service:
              name: demo-reads-service
              port:
                number: 80
          pathType: ImplementationSpecific
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: demo-ingress
```
