# Setting up ExternalDNS for PowerDNS

## Prerequisites

The provider has been written for and tested against [PowerDNS](https://github.com/PowerDNS/pdns) v4.1.x and thus requires **PowerDNS Auth Server >= 4.1.x**

PowerDNS provider support was added via [this PR](https://github.com/kubernetes-incubator/external-dns/pull/373), thus you need to use external-dns version >= v0.5

The PDNS provider expects that your PowerDNS instance is already setup and
functional. It expects that zones, you wish to add records to, already exist
and are configured correctly. It does not add, remove or configure new zones in
anyway.

## Feature Support

The PDNS provider currently does not support:

1. Dry running a configuration is not supported.
2. The `--domain-filter` flag is not supported.

## Deployment

Deploying external DNS for PowerDNS is actually nearly identical to deploying
it for other providers. This is what a sample `deployment.yaml` looks like:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: deploy-external-dns
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      # Only use if you're also using RBAC
      # serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:v0.5.0
        args:
        - --source=service # or ingress or both
        - --provider=pdns
        - --pdns-server={{ pdns-api-url }}
        - --pdns-api-key={{ pdns-http-api-key }}
        - --txt-owner-id={{ owner-id-for-this-external-dns }}
        - --log-level=debug
        - --interval=30s
```

## RBAC

If your cluster is RBAC enabled, you also need to setup the following, before you can run external-dns:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
```

## Testing and Verification

**Important!**: Remember to change `example.com` with your own domain throughout the following text.

Spin up a simple "Hello World" HTTP server with the following spec (`kubectl apply -f`):

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: echo
spec:
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: hashicorp/http-echo
        name: echo
        ports:
        - containerPort: 5678
        args:
          - -text="Hello World"
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  annotations:
    external-dns.alpha.kubernetes.io/hostname: echo.example.com
spec:
  selector:
    app: echo
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```
**Important!**: Don't run dig, nslookup or similar immediately (until you've
confirmed the record exists). You'll get hit by [negative DNS caching](https://tools.ietf.org/html/rfc2308), which is hard to flush.

Run the following to make sure everything is in order:

```bash
$ kubectl get services echo
$ kubectl get endpoints echo
```

Make sure everything looks correct, i.e the service is defined and recieves a
public IP, and that the endpoint also has a pod IP.

Once that's done, wait about 30s-1m (interval for external-dns to kick in), then do:
```bash
$ curl -H "X-API-Key: ${PDNS_API_KEY}" ${PDNS_API_URL}/api/v1/servers/localhost/zones/example.com. | jq '.rrsets[] | select(.name | contains("echo"))'
```

Once the API shows the record correctly, you can double check your record using:
```bash
$ dig @${PDNS_FQDN} echo.example.com.
```

