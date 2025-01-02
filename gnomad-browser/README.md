# gnomAD Browser

## Deployment

### Prerequisites

- It is assumed that there is a configmap in the environment called `proxy-ips` that contains the IP addresses of any load balancers that might forward traffic before it reaches the service.

  This is created for you if you deployed your GKE cluster using the gnomad-browser-infra module. If you need to create it by hand, you can do so with the following command.

  ```sh
    kubectl create configmap proxy-ips --from-literal=ips="your.load.balacer.ip,127.0.0.1"
  ```

  If you're running something in development and don't necessarily care if remote/user IP addresses are reported correctly, a value of `"127.0.0.1"` is sufficient.

- It is assumed that there is a kubernetes service account called `gnomad-api` that is bound to an IAM identity that can read from the gnomad JSON cache bucket in GCS.

  This is created for you if you deployed your GKE cluster using the gnomad-browser-infra module. If you need to create it by hand, you can do so with the following command.

  ```sh
    kubectl create serviceaccount gnomad-api
  ```

  As a TODO for the future: we should create this service account as an artifact of the deployment, when google makes it more ergonomic to grant IAM permissions directly to kubernetes service accounts.

### Production

The production deployment is comprised of three kustomize layers: base, prod, and {blue,green}. The base config is the minimal browser deployment, with settings appropriate for all types of browser deployments. The prod layer is default settings that should be applied to all production deployments. The blue/green layer applies the blue/green naming suffixes, labels, and any deployment-specific items.

Finally, there is a "prod-deflector" service definition, which is used to point the ingress at whichever blue/green deployment is the active production deployment.

You can render and diff each of these deployments against production by cd'ing into the gnomad-browser folder, and running:

```sh
  # ensure that your kubectl context is pointed at the appropriate cluster
  kustomize build blue
  kustomize build blue | kubectl diff -f -
```

#### Making changes to prod

Docker image updates for the blue/green service are automated via cloudbuild. Upon merging a PR in the gnomad-browser repo, a cloudbuild job will:

1. build and push new browser and API docker images
2. query the production cluster to determine if it should update blue or green
3. update the image tag in the target environment
4. Push the change to the deployment in this repository

Once that deployment is complete, you can test/demo the new blue/green environment until you're satisfied that it is ready. At that time, you can run the cloudbuild job in the exac-gnomad project that flips the deflector service between blue and green (TODO: Link to the job here). Rollbacks can be performed by re-running the deflector switch cloudbuild job.

### Development / Demo

For ease of use, there is a `example-demo` template and a `make` task that can be run to generate a new demo. The make task takes two parameters:

1. demo_name: The suffix you want to apply to your demo
2. docker_tag: The docker image tag you want to apply to your demo deployment. **The docker image will reference the `gnomadev` docker registry.**

The make file will create a new folder with your kustomization in it, which you can then tweak as needed.

```sh
cd gnomad-browser
make demo demo_name=my-demo docker_tag=abcdef1
```

If you would like to generate a new deployment manually, cd to the gnomad-browser directory, and init a new kustomization layer that inherits from the demo layer (or, optionally, a different layer of your choice):

```sh
  cd gnomad-browser
  mkdir my-demo && cd my-demo
  kustomize init --resources ../demo
```

In the resulting kustomization.yaml file, you can add necessary patches for your deployment. See below for examples of common patches.

## Common Patches

### Adding a name suffix to your deployment

```yaml
# in kustomization.yaml, add:
nameSuffix: "-mydemo"

# you also need a replacement defined in your layer that adjusts the API_URL env var:
replacements:
  - source:
      fieldPath: metadata.name
      kind: Service
      name: gnomad-api
      version: v1
    targets:
      - fieldPaths:
          - spec.template.spec.containers.0.env.0.value
        options:
          delimiter: /
          index: 2
        select:
          group: apps
          kind: Deployment
          name: gnomad-browser
          version: v1
```

### Overriding the image tags

```yaml
# in kustomization.yaml, add something like this to override docker tags:
images:
  - name: us-docker.pkg.dev/exac-gnomad/gnomad/gnomad-api
    newTag: 'my-fancy-api-image'
  - name: us-docker.pkg.dev/exac-gnomad/gnomad/gnomad-browser
    newTag: 'my-fancy-browser-image'


# If you also need to override the docker repository, use newName:
images:
  - name: us-docker.pkg.dev/exac-gnomad/gnomad/gnomad-api
    newName: us-docker.pkg.dev/dev-repo/gnomad/my-gnomad-api
    newTag: 'my-fancy-api-image'
  - name: us-docker.pkg.dev/exac-gnomad/gnomad/gnomad-browser
    newName: us-docker.pkg.dev/dev-repo/gnomad/my-gnomad-browser
    newTag: 'my-fancy-browser-image'
```

### Changing resource limits and requests

```yaml
# in kustomization.yaml, in the patches list:
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: gnomad-api
      spec:
        template:
          spec:
            containers:
              - name: app
                resources:
                  requests:
                    cpu: '1'
                    memory: '11Gi'
                  limits:
                    cpu: '2'
                    memory: '12Gi'
```
