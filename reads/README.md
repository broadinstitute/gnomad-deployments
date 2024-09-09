# gnomAD Reads service

## Deployment

### Prerequisites

- It is assumed that there is a configmap in the environment called `proxy-ips` that contains the IP addresses of any load balancers that might forward traffic before it reaches the service.

  This is created for you if you deployed your GKE cluster using the gnomad-browser-infra module. If you need to create it by hand, you can do so with the following command.

  ```sh
    kubectl create configmap proxy-ips --from-literal=ips="your.load.balacer.ip,127.0.0.1"
  ```

  If you're running something in development and don't necessarily care if remote/user IP addresses are reported correctly, a value of `"127.0.0.1"` is sufficient.

- The reads service requires a persistent disk to be provisioned with data. You can then reference the name of that disk in the deployment manifest, with the `readviz` volume in the Deployment manifest. See [Common patches](#common-patches) for an example of how to override the persistent disk name for testing. If you are updating a production environment, you should set the default persistent disk name in the Deployment manifest in `base/`.

### Production

To update the production reads service, update the image tag in the `bluegreen/kustomization.yaml` file, and open a PR. After review and merge, the service will automatically deploy to a preview environment. You can test/verify the preview deployment by setting your reads API URL to `https://gnomad.broadinstitute.org/reads-preview`.

After verifying the preview deployment, you can promote the preview deployment to production using the `kubectl argo rollouts promote` command, or by clicking the "Promote" button in the Argo CD UI.

### Development

To deploy a standalone development version of the reads service, you can run the following command, you can deploy the base configuration without modifications by running:

```sh
# First, ensure that your kubectl context is set to the correct cluster
kustomize build base | kubectl apply -f -
```

If you need to change anything about the base deployment, such as the persistent disk name or adding a name prefix or suffix, you can create a new folder at the same level as `base/`, and then use kustomize to create a new overlay:

```sh
mkdir my-reads && cd my-reads
kustomize init --resource=../base
```

Then you can add the necessary patches to your kustomization. See the examples below for common patches.

## Common patches

### Overriding the default persistent data volume:

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: gnomad-reads
      spec:
        template:
          spec:
            volumes:
              - name: readviz
                gcePersistentDisk:
                  pdName: test-readviz-data
```

### Adding a name suffix to the deployment:

```yaml
# in kustomization.yaml, add:
nameSuffix: "-test"
```

### Overriding the default images:

```yaml
# in kustomization.yaml, add:
images:
  - name: gnomad-reads-server
    newName: us-docker.pkg.dev/gnomadev/gnomad/gnomad-reads-server
    newTag: 'my-fancy-reads-image'
  - name: gnomad-reads-api
    newName: us-docker.pkg.dev/gnomadev/gnomad/gnomad-reads-api
    newTag: 'my-fancy-reads-api-image'
