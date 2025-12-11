# The ingress-nginx Retirement (Part 2): Migrating to Traefik

In [Part 1](https://www.linkedin.com/pulse/ingress-nginx-retirementpart-1-gerard-van-engelen-mliqe/?trackingId=3XyZk3y7QSmPKj3PSrzl2g%3D%3D), I covered why ingress-nginx is retiring and evaluated three replacement options: Traefik, Istio, and Cilium. Today, I'll walk you through actually migrating to Traefik, starting with a lift-and-shift approach using the Ingress API, then moving to the Gateway API.

This guide is hands-on. We'll start with a KIND cluster running ingress-nginx, migrate to Traefik's Ingress Controller, and then transition to Gateway API.

## Why Start with Traefik?

As I mentioned in Part 1, Traefik offers the smoothest migration path. This means you can migrate quickly with minimal code changes, then progressively adopt Gateway API features at your own pace.

## What do you need

For this tutorial, you'll need:

- Podman, Docker, or something else you like
- kubectl
- KIND
- Helm

## Part 1: Setting Up the Baseline(ingress-nginx)

Let's start by creating a KIND cluster with ingress-nginx, similar to what you're probably running in production.

### Create the KIND Cluster

Create a file named `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

Create the cluster:

```bash
kind create cluster --config kind-config.yaml --name ingress-migration
```

### Install ingress-nginx

First, add the ingress-nginx Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Create a values file for ingress-nginx (`ingress-nginx-values.yaml`):

```yaml
controller:
  hostPort:
    enabled: true
  service:
    type: NodePort

  metrics:
    enabled: true
    serviceMonitor:
      enabled: false

  publishService:
    enabled: false

  extraArgs:
    v: 2

podSecurityPolicy:
  enabled: false
```

Install ingress-nginx:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values ingress-nginx-values.yaml
```

Wait for it to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=ingress-nginx \
  --timeout=90s
```

### Deploy a Sample Application

Create `demo-app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-service
  namespace: demo
spec:
  selector:
    app: echo-server
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

Apply it:

```bash
kubectl apply -f demo-app.yaml
```

### Test the Baseline Setup

Add `echo.local` to your `/etc/hosts`:

```bash
echo "127.0.0.1 echo.local" | sudo tee -a /etc/hosts
```

Test it:

```bash
curl http://echo.local/api/hello
```

You should see a JSON response from the echo server. We've got a working ingress-nginx setup, nothingn fancy I know.

## Part 2: Lift-and-Shift Migration to Traefik

Now let's replace ingress-nginx with Traefik's Ingress Controller while keeping the same Ingress API resources.

### Step 1: Remove ingress-nginx

On your production environment, you can easily create a second load balancer and connect that to Traefik. But on KIND you cannot do it, so we'll delete the ingress-nginx first.

```bash
helm uninstall ingress-nginx -n ingress-nginx
```

### Step 2: Install Traefik

First, add the Traefik Helm repository:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Create a values file for Traefik (`traefik-values.yaml`):

```yaml
ingressRoute:
  dashboard:
    enabled: true

service:
  type: LoadBalancer

providers:
  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
    ingressClass: traefik
    nativeLBByDefault: true
    allowEmptyServices: true
    namespaces: []

ports:
  web:
    port: 8000
    exposedPort: 80
    protocol: TCP
    hostPort: 80
  websecure:
    port: 8443
    exposedPort: 443
    protocol: TCP
    hostPort: 443

logs:
  general:
    level: INFO
  access:
    enabled: true
```

Install Traefik:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --values traefik-values.yaml
```

Wait for Traefik to be ready:

```bash
kubectl wait --namespace traefik \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=traefik \
  --timeout=90s
```

### Step 3: Update the Ingress Resource

While writing my first post, I read a lot about Traefik being able to migrate with ingress-nginx annotations. But while creating this blog I found out that Traefik v3 removed automatic ingress-nginx annotation translation.

This makes migrating a little more tedious. But because you can move service by service, I think it remains doable.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-service
  namespace: demo
spec:
  selector:
    app: echo-server
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: demo
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: demo-stripprefix@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: echo.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: echo-service
                port:
                  number: 80
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: stripprefix
  namespace: demo
spec:
  stripPrefix:
    prefixes:
      - /api
```

Apply it:

```bash
kubectl apply -f demo-app.yaml
```

You'll see that we removed the old ingress-nginx annotations and replaced them to use traefiks middlewares. In this case we've created a middleware that removes tha `/api` from the call.

### Step 4: Test the Migration

```bash
curl http://echo.local/api/hello
```

You should get the same response as before. The application hasn't changed, but now Traefik is handling the routing.

## Part 3: Migrating to Gateway API

Now that we have Traefik running, let's migrate to Gateway API. This is where things get interesting, and more powerful.

### Step 1: Enable Traefik Gateway API

First we'll enable the Gateway API on our cluster.

update `gateway-infrastructure.yaml`:

```yaml
ingressRoute:
  dashboard:
    enabled: true

service:
  type: LoadBalancer

providers:
  kubernetesIngress:
    enabled: false
    allowExternalNameServices: true
    ingressClass: traefik
    nativeLBByDefault: true
    allowEmptyServices: true
    namespaces: []
  kubernetesGateway:
    enabled: true

ports:
  web:
    port: 8000
    exposedPort: 80
    protocol: TCP
    hostPort: 80
  websecure:
    port: 8443
    exposedPort: 443
    protocol: TCP
    hostPort: 443

logs:
  general:
    level: INFO
  access:
    enabled: true
```

We'll need to delete the old controller because of port limitations in KIND.

```bash
k get pods -n traefik
```

Now copy the name of the running pod and delete that one:

```bash
k delete pod {podname} -n traefik
```

If you get the pods again, the previously pending pod should be running.

### Step 2: Create the Gateway Resources

Gateway API introduces a clear separation of concerns. Let's create the infrastructure layer first.

Create `traefik-values.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: shared-gateway
spec:
  controllerName: traefik.io/gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: http
      protocol: HTTP
      port: 8000
      allowedRoutes:
        namespaces:
          from: All
```

NOTE: you could use the default gateway class created by Traefik's helm chart but for the sake of being complete I've added a "custom" class.

Apply it:

```bash
kubectl apply -f gateway-infrastructure.yaml
```

Check the Gateway status:

```bash
kubectl get gateway -n traefik
```

You should see the Gateway is `Accepted` and `Programmed`.

### Step 3: Create HTTPRoute for the Application

Now let's create the application routing layer. This is where application teams define their routes.

Create `httproute-echo.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
  namespace: demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: traefik
  hostnames:
    - echo.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: echo-service
          port: 80
```

Apply it:

```bash
kubectl apply -f httproute-echo.yaml
```

### Step 4: Test Gateway API

```bash
curl http://echo.local/api/hello
```

Same response, but now using Gateway API!

### Understanding the Gateway API Model

Let's see what we just created:

**GatewayClass**
Think of this as the "driver" for your infrastructure. It tells Kubernetes which controller(Traefik, in our case) should manage these Gateways.

**Gateway**
This is the actual load balancer infrastructure. In production, this would create a cloud load balancer. In our KIND setup, it uses NodePorts. Platform teams typically manage this resource.

**HTTPRoute**
This is where application teams define their routing rules. It's similar to an Ingress resource but more feature rich and expressive.

### Advanced Gateway API Features

Gateway API offers a lot of improvements over the Ingress API. Let's explore some powerful features.

#### Feature 1: Traffic Splitting

Let's deploy a v2 of our echo server and split traffic:

Update `demo-app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-service
  namespace: demo
spec:
  selector:
    app: echo-server
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server-v2
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
      version: v2
  template:
    metadata:
      labels:
        app: echo-server
        version: v2
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
            - name: ENVIRONMENT
              value: "v2"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-service-v2
  namespace: demo
spec:
  selector:
    app: echo-server
    version: v2
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: demo
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: demo-stripprefix@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: echo.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: echo-service
                port:
                  number: 80
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: stripprefix
  namespace: demo
spec:
  stripPrefix:
    prefixes:
      - /api
```

Apply it:

```bash
kubectl apply -f demo-app.yaml
```

Now update the HTTPRoute to split traffic 80/20:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
  namespace: demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: traefik
  hostnames:
    - echo.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: echo-service
          port: 80
          weight: 80
        - name: echo-service-v2
          port: 80
          weight: 20
```

Apply it:

```bash
kubectl apply -f httproute-echo.yaml
```

And test it multiple times:

```bash
for i in {1..10}; do curl -s http://echo.local/api/hello; echo; done
```

You'll see approximately 80% v1 responses and 20% v2 responses. It's important to note that weighted loadbalancing doesn't gaurantee a hard 80/20 split.

#### Feature 2: Header-Based Routing

Gateway API makes header-based routing simple:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route-beta
  namespace: demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: traefik
  hostnames:
    - echo.local
  rules:
    # Beta users get v2
    - matches:
        - path:
            type: PathPrefix
            value: /api
          headers:
            - name: X-Beta-User
              value: "true"
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: echo-service-v2
          port: 80
    # Everyone else gets v1
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: echo-service
          port: 80
```

Test it:

```bash
# Regular users get v1
curl http://echo.local/api/hello

# Beta users get v2
curl -H "X-Beta-User: true" http://echo.local/api/hello
```

#### Feature 3: Request Mirroring

Mirror production traffic to a test environment:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route-mirror
  namespace: demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: traefik
  hostnames:
    - echo.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
        - type: RequestMirror
          requestMirror:
            backendRef:
              name: echo-service-v2
              port: 80
      backendRefs:
        - name: echo-service
          port: 80
```

Now all requests go to v1, but they're also mirrored to v2 for testing.

## Comparison: Ingress API vs Gateway API

The Gateway API version is more verbose, but it's also:

- More explicit, no hidden behaviors in annotations
- More portable, not controller-specific
- More powerful, native support for advanced routing
- Better separated, platform vs application configurations

## Migration Strategy for Production

Moving to Gateway API involves more than replacing one controller with another. It introduces new recources and requires updating everything, including training developers about the new architecture.

If you're timebound, my tip would be to do a lift and shift from ingress-nginx to Traefik. After that, you can take your time and plan the Gateway API migration. My high level plan would be:

### 1. Lift-and-Shift

1. Install Traefik alongside ingress-nginx
2. Change `ingressClassName` for non-critical applications
3. Monitor for issues
4. Roll back if needed (just change the class back)

### 2. Validate and Expand

1. Migrate more applications
2. Identify and fix annotation compatibility issues
3. Create Traefik Middlewares for custom behaviors
4. Update runbooks and documentation

### 3. Gateway API Adoption

1. Create GatewayClass and Gateway resources
2. Convert Ingress resources to HTTPRoutes progressively
3. Start with simple routes, then complex ones

### 4. Cleanup

1. Remove ingress-nginx
2. Clean up unused Ingress resources
3. Update monitoring and alerting
4. Train team on Gateway API

## Monitoring Your Migration

Key metrics to watch:

- Request latency (compare before/after)
- Error rates (should remain stable)
- Request distribution (verify traffic splitting)
- Gateway status (ensure it stays Programmed)

Unfortunately, Traefik and ingress-nginx expose different metrics. This means you should update dashboards, alerts, etc.

If you're using Grafana there is a nice official dashboard ready to import [here](https://grafana.com/grafana/dashboards/17347-traefik-official-kubernetes-dashboard/).

## What's Next

In Part 3, I'll cover migrating to Cilium. This is a more complex migration since Cilium requires replacing your entire CNI, but it offers incredible performance benefits through eBPF.

In Part 4, we'll tackle Istio, which brings service mesh capabilities along with ingress functionality.

## Cleanup

To clean up your KIND cluster:

```bash
kind delete cluster --name ingress-migration
```

---

_All code examples from this post are available in my [GitHub repository](https://github.com/GVengelen/ingress-traefik-migration)._
