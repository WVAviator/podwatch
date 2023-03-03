# PodWatch

Quickly and easily monitor errors within a Kubernetes cluster. PodWatch provides a simple and easy way to stream all errors coming from your cluster into the PodWatch Service or customer service developed by you. You can generate notifications, trigger webhooks, and view all your errors in one convenient place.

## Getting started

### Prerequisites

- A running Kubernetes cluster
- Access to apply new configurations to the cluster

### Setup

Setup is quick and easy, taking less than five minutes. First, head to the PodWatch website and create an account. Add a new cluster to your account, and get your cluster ID and secret. If you do not wish to use the PodWatch web service, you can skip down to the [Custom Server](#custom-server) instructions.

Next, create the Kubernetes YAML config files for PodWatch. You will need a Deployment, ServiceAccount, ClusterRole, and ClusterRoleBinding. Provided below are some examples that can be copied and added into your cluster. Once you create these files, you can simply add them to your cluster with `kubectl apply -f podwatch`, where `podwatch` is a directory containing all four config files (alternatively you can apply them one-at-a-time if you choose).

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podwatch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podwatch
  template:
    metadata:
      labels:
        app: podwatch
    spec:
      containers:
        - name: podwatch
          image: wvaviator/podwatch:latest
          env:
            - name: PODWATCH_CLIENT_ID
              value: 'your cluster ID here'
            - name: PODWATCH_CLIENT_SECRET
              value: 'your cluster secret here'
      serviceAccountName: podwatch-serviceaccount
```

_Note: if you do not wish to include your client ID and secret directly in the YAML file and would prefer other ways of managing secrets, check out the [Kubernetes documentation on secrets](https://kubernetes.io/docs/concepts/configuration/secret/)._

#### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: podwatch-serviceaccount
```

#### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: default
  name: podwatch-role
rules:
  - apiGroups: ['']
    resources: ['events']
    verbs: ['get', 'watch', 'list']
```

#### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: podwatch-role-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: podwatch-serviceaccount
    namespace: default
roleRef:
  kind: ClusterRole
  name: podwatch-role
  apiGroup: rbac.authorization.k8s.io
```

### Custom Server

Optionally, if you prefer not to use the PodWatch web service, you can still set up your own custom server with a webhook that will be called by PodWatch upon receiving errors from the cluster. Within this custom server, you can set up custom actions, messages, notifications, or any other custom responses to cluster errors.

To set up a custom server, you can use the following deployment config (which is similar to the config example above, with different env variables). You will still need to create a ServiceAccount, ClusterRole, and ClusterRoleBinding as in the provided examples above.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podwatch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podwatch
  template:
    metadata:
      labels:
        app: podwatch
    spec:
      containers:
        - name: podwatch
          image: wvaviator/podwatch:latest
          env:
            - name: PODWATCH_CUSTOM_SERVER_URL
              value: https://my.custom.url/errors
      serviceAccountName: podwatch-serviceaccount
```

The PODWATCH_CUSTOM_SERVER_URL should be your endpoint to receive errors from PodWatch. This endpoint will be called by PodWatch with a POST request, with a request body that looks like this:

```json
{
  "name": "podwatch-984hv9w8hf-384hf.c3niwfjr8h34f",
  "reason": "FailedKillPod",
  "message": "error killing pod: failed to \"KillContainer\" for \"podwatch\" with KillContainerError: \"rpc error: code = Unknown desc = Error response from daemon: No such container\"",
  "type": "Warning",
  "firstTimestamp": "2023-03-03T17:28:02Z",
  "lastTimestamp": "2023-03-03T17:28:02Z",
  "count": 1,
  "nativeEvent": { "type": "DELETED", "object": { ... } }
}
```

The `nativeEvent` property represents the original event generated by Kubernetes. Use this if you need to access specific information about the resource that caused the error, metadata about the error or associated containers, and other specific information not included in the top-level summary fields.
