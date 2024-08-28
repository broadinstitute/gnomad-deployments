# gnomAD Reads service

## Deployment

### Prerequisites

- It is assumed that there is a configmap in the environment called `proxy-ips` that contains the IP addresses of any load balancers that might forward traffic before it reaches the service.

  This is created for you if you deployed your GKE cluster using the gnomad-browser-infra module. If you need to create it by hand, you can do so with the following command.

  ```sh
    kubectl create configmap proxy-ips --from-literal=ips="your.load.balacer.ip,127.0.0.1"
  ```

- The reads service requires a persistent disk to be provisioned with data. You can then reference the name of that disk in the deployment manifest, with the `readviz` volume in the Deployment manifest. See [Common patches](#common-patches) for an example of how to override the persistent disk name for testing. If you are updating a production environment, you should set the default persistent disk name in the Deployment manifest in `base/`.

If you're running something in development and don't necessarily care if remote/user IP addresses are reported correctly, a value of `"127.0.0.1"` is sufficient.

### Production

To update the production reads service, update the image tag in the `bluegreen/kustomization.yaml` file, and open a PR. After review and merge, the service will automatically deploy to a preview environment. You can test/verify the preview deployment by setting your reads API URL to `https://gnomad.broadinstitute.org/reads-preview`.

After verifying the preview deployment, you can promote the preview deployment to production using the `kubectl argo rollouts promote` command, or by clicking the "Promote" button in the Argo CD UI.

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
        replicas: 0
        template:
          spec:
            volumes:
              - name: readviz
                gcePersistentDisk:
                  pdName: readviz-empty-test
```
