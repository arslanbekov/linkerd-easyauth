# Linkerd EasyAuth Extension

## Motivation
Simplify the Linkerd Authorization Policies management according to [the article](https://itnext.io/a-practical-guide-for-linkerd-authorization-policies-6cfdb50392e9) by giving a bunch of predefined policies and opinionated structures.

Special checkers to find obsolete resources and misconfigurations, plus ultra-fast `authz` command implementation (up to 10x faster than original one).

## Supported versions
| Linkerd Version | EasyAuth Version |
|-----------------|------------------|
| 2.11.x          | 0.1.0 - 0.4.0    |
| 2.12.x          | \>= 0.5.0        |

New `AuthorizationPolicy` is supported since `0.6.0`.  New `HTTPRoute` is supported since `0.8.0`

## How to use it

## CLI
Grab latest binaries from the releases page: https://github.com/aatarasoff/linkerd-easyauth/releases.

### Usage
```
linkerd easyauth [COMMAND] -n <namespace> [FLAGS]
```

### Supported commands
- `authcheck`: checks for obsolete `Server` and policies resources like `ServerAuthorization`, `AuthorizationPolicy`, `MeshTLSAuthentication`, `NetworkAuthentication`, and `HTTPRoute`, checks that PODs ports have `Server` resource
- `list`: list of Pods that were injected by `linkerd.io/easyauth-enabled: true` annotation (more information below)
- `authz`: fast implementation for fetch the list authorization policies for a resource (use caching)

## Helm chart
Install the helm chart with injector and policies:
```
> kubectl create ns linkerd-easyauth

# Edit namespace and add standard linkerd annotations

> helm repo add linkerd-easyauth https://aatarasoff.github.io/linkerd-easyauth
> helm install -n linkerd-easyauth linkerd-easyauth linkerd-easyauth/linkerd-easyauth --values your_values.yml
```

### What the helm chart provides
- Injector that adds `linkerd.io/easyauth-enabled: true` label for all meshed pods (you can limit namespaces via helmchart)
- `Server` in terms of Linkerd authorization policies for `linkerd-admin-port`
- `AuthorizationPolicy` resources that provides basic allow policies for ingress, Linkerd itself, and monitoring

### What the helm chart does not provide
Because the `Server` should be one per service per port, we can define the server for the linkerd proxy admin port only.
For each port that should be used by other pods, or Linkerd you should add the server definition manually:
```
---
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  namespace: <app-namespace>
  name: <app-server-name>
  labels:
    linkerd.io/server-type: common
spec:
  podSelector:
    matchLabels:
      <app-label>: <app-unique-value>
  port: <my-port-name>
``` 

### Important Values
#### Meshed Apps Namespaces
Because all `AuthorizationPolicy` policies are Namespaced scope then we should add common policies to each namespace with our apps:
```
meshedApps:
  namespaces:
    - hippos
    - elephants
```

#### Cluster Network Common Policy
In case of using route-based policy you should authorize requests for passing probes by adding app-specific `HTTPRoute` and policies for it for each app:
```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: cool-app-health-check-allow
  namespace: cool-ns
spec:
  targetRef:
    group: policy.linkerd.io
    kind: HTTPRoute
    name: cool-app-health-check
  requiredAuthenticationRefs:
    - name: cluster-network-authn
      kind: NetworkAuthentication
      group: policy.linkerd.io
```

The Helm chart generates NetworkAuthentication with name `cluster-network-authn` to authorize cluster network requests.

You should explicitly provide cluster network or authorize kubelet only. It depends on the K8s implementation you are using and could be setup via `clusterNetwork` section in the values.

#### Kubelet CIDR
> **⚠ WARNING: 2.11.x only**  

Because of [the issue](https://github.com/linkerd/linkerd2/issues/7050), in 2.11.x version of Linkerd you should explicitly provide CIDR for kubelet.
It depends on the K8s implementation you are using.

There are two possibility. If you can define CIDR precisely then you can use it
```
  kubelet:
    cidr:
      - cidr: 10.164.0.0/20
```

If you cannot do it, but you have GKE-like pattern then you can define octets and ranges for generation the bunch of `/32` CIDR:
```
  kubelet:
    cidr: []
    # generate by pattern octet0:{low1-high1}:{low2-high2}:octet3 (10.169.150.1)
    generator:
      octet0: 10
      low1: 168
      high1: 172
      low2: 0
      high2: 256
      octet3: 1
```
