# gnomAD Reads service

## Deployment

### Prerequisites

- It is assumed that there is a configmap in the environment called `proxy-ips` that contains the IP addresses of any load balancers that might forward traffic before it reaches the service.

  This is created for you if you deployed your GKE cluster using the gnomad-browser-infra module. If you need to create it by hand, you can do so with the following command.

  ```sh
    kubectl create configmap proxy-ips --from-literal=ips="your.load.balacer.ip,127.0.0.1"
  ```

If you're running something in development and don't necessarily care if remote/user IP addresses are reported correctly, a value of `"127.0.0.1"` is sufficient.

### Common patches

#### Overriding the default persistent data volume:

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
