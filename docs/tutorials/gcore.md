# Setting up ExternalDNS for Services on G-Core Labs

This tutorial describes how to setup ExternalDNS for use within a
Kubernetes cluster using G-Core Labs DNS.

Make sure to use **>=0.10** version of ExternalDNS for this tutorial.

## Creating a zone with G-Core Labs DNS

If you are new to G-Core Labs, we recommend you first read the following
instructions for creating a zone.

[Creating a zone using the G-Core Labs web console](https://dns.gcorelabs.com/zones)

[Creating a zone using the G-Core Labs API](https://dnsapi.gcorelabs.com/docs#operation/CreateZone)

## Creating G-Core Labs API key

You first need to create a permanent API token.

Using the [G-Core Labs documentation](https://gcorelabs.com/support/articles/360018625617/) you will have your `permanent API token`

## Deploy ExternalDNS

Connect your `kubectl` client to the cluster with which you want to test ExternalDNS, and then apply one of the following manifest files for deployment:

### Manifest (for clusters without RBAC enabled)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.10.0
        args:
         - --source=service # ingress is also possible
         - --domain-filter=example.com # (optional) limit to only example.com domains; change to match the zone created above.
         - --provider=gcore
        env:
         - name: GCORE_PERMANENT_API_TOKEN
           value: "YOUR_GCORE_PERMANENT_API_TOKEN"
```

### Manifest (for clusters with RBAC enabled)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.7
        args:
          - --source=service # ingress is also possible
          - --domain-filter=example.com # (optional) limit to only example.com domains; change to match the zone created above.
          - --provider=gcore
        env:
          - name: GCORE_PERMANENT_API_TOKEN
            value: "YOUR_GCORE_PERMANENT_API_TOKEN"
```

## Deploying an Nginx Service

Create a service file called 'nginx.yaml' with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: example.com
    external-dns.alpha.kubernetes.io/ttl: "120" #optional
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**A note about annotations**

Verify that the annotation on the service uses the same hostname as the G-Core Labs DNS zone created above. The annotation may also be a subdomain of the DNS zone (e.g. 'www.example.com').

The TTL annotation can be used to configure the TTL on DNS records managed by ExternalDNS and is optional. If this annotation is not set, the TTL on records managed by ExternalDNS will default to 10.

ExternalDNS uses the hostname annotation to determine which services should be registered with DNS. Removing the hostname annotation will cause ExternalDNS to remove the corresponding DNS records.

### Create the deployment and service

```
$ kubectl create -f nginx.yaml
```

Depending on where you run your service, it may take some time for your cloud provider to create an external IP for the service. Once an external IP is assigned, ExternalDNS detects the new service IP address and synchronizes the G-Core Labs DNS records.

## Verifying G-Core Labs DNS records

Use the G-Core Labs web console or API to verify that the A record for your domain shows the external IP address of the services.

## Cleanup

Once you successfully configure and verify record management via ExternalDNS, you can delete the tutorial's example:

```
$ kubectl delete -f nginx.yaml
$ kubectl delete -f externaldns.yaml
```