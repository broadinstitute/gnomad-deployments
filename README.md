# gnomAD Deployment definitions

This repository houses the deployment definitions for the gnomAD project. Production deployments are generally managed by ArgoCD, and are applied to a GKE cluster either automatically, or via the ArgoCD web UI, based on what is appropriate for that particular deployment.

## Kustomize

For the most part, these deployments are defined with [Kustomize](https://kustomize.io/). Note that kustomize can be used in two ways:

1. As a standalone tool, where you run `kustomize build` to render the final configuration, and then pipe that to `kubectl apply -f -` to apply it to the cluster.
2. As part of kubectl, with the `-k` flag.

We typically assume the use of the standalone tool, since the one bundled with kubectl is sometimes out of date, and doesn't support all the features we are using. You may find that the kubectl version works fine, and it's OK to use it in those cases. However, if you run into things that don't work, try using the standalone tool.

The standalone tool can be installed on macOS with `brew install kustomize`, or on other platforms by downloading the binary from the [releases page](https://github.com/kubernetes-sigs/kustomize/releases).
