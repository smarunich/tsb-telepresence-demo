# Using Telepresence with Tetrate Service Bridge

# Prerequisites

1. Telepresence binary
2. TSB control plane installed to cluster
3. kubectl
4. Working ingress controller

# Steps

1. Create a target namespace tsb-telepresence-demo
  ```
  kubectl create ns tsb-telepresence-demo
  kubectl label namespace tsb-telepresence-demo istio-injection=enabled --overwrite
  ```
2. Apply TSB configuration: replace <cluster> with the destination k8s cluster name before applying
  ```
  cat <<EOF > tsb-mp-configuration.yaml
  ---
  apiVersion: api.tsb.tetrate.io/v2
  kind: Tenant
  metadata:
    displayName: tsb-telepresence-demo
    name: tsb-telepresence-demo
    organization: tetrate
  spec:
    displayName: tsb-telepresence-demo
  ---
  apiVersion: rbac.tsb.tetrate.io/v2
  kind: TenantAccessBindings
  metadata:
    displayName: tsb-telepresence-demo
    organization: tetrate
    tenant: tsb-telepresence-demo
  spec:
    allow:
    - role: rbac/admin
      subjects:
      - user: admin
  ---
  apiVersion: api.tsb.tetrate.io/v2
  kind: Workspace
  metadata:
    name:  tsb-telepresence-demo-ws
    organization: tetrate
    tenant: tsb-telepresence-demo
  spec:
    displayName: tsb-telepresence-demo-ws
    namespaceSelector:
      names:
      - <cluster>/tsb-telepresence-demo
  ---
  apiVersion: rbac.tsb.tetrate.io/v2
  kind: WorkspaceAccessBindings
  metadata:
    name:  tsb-telepresence-demo
    organization: tetrate
    tenant: tsb-telepresence-demo
    workspace: tsb-telepresence-demo-ws
  spec:
    allow:
    - role: rbac/admin
      subjects:
      - user: admin
  ---
  apiVersion: gateway.tsb.tetrate.io/v2
  kind: Group
  metadata:
    organization: tetrate
    tenant: tsb-telepresence-demo
    workspace: tsb-telepresence-demo-ws
    name: tsb-telepresence-demo-gw
  spec:
    displayName: tsb-telepresence-demo-gw
    namespaceSelector:
      names:
        - <cluster>/tsb-telepresence-demo
    configMode: BRIDGED
  ---
  apiVersion: traffic.tsb.tetrate.io/v2
  kind: Group
  metadata:
    organization: tetrate
    tenant: tsb-telepresence-demo
    workspace: tsb-telepresence-demo-ws
    name: tsb-telepresence-demo-traffic
  spec:
    displayName: tsb-telepresence-demo-traffic
    namespaceSelector:
      names:
        - <cluster>/tsb-telepresence-demo
    configMode: BRIDGED
  ---
  apiVersion: security.tsb.tetrate.io/v2
  kind: Group
  metadata:
    organization: tetrate
    tenant: tsb-telepresence-demo
    workspace: tsb-telepresence-demo-ws
    name: tsb-telepresence-demo-security
  spec:
    displayName: tsb-telepresence-demo-security
    namespaceSelector:
      names:
        - <cluster>/tsb-telepresence-demo
    configMode: BRIDGED
  EOF
  tctl apply -f tsb-mp-configuration.yaml
  cat <<EOF > tsb-gw-configuration.yaml
  ---
  apiVersion: install.tetrate.io/v1alpha1
  kind: IngressGateway
  metadata:
    name: tsb-telepresence-demo-gw
    namespace: tsb-telepresence-demo
  spec:
    kubeSpec:
      service:
        type: LoadBalancer
  EOF
  kubectl apply -f tsb-gw-configuration.yaml
  ```

2. Prepare local environment, perform steps from 1 to 2: https://www.telepresence.io/docs/latest/quick-start/qs-node/

3. Install a sample Node.js application
  ```
  kubectl apply -n tsb-telepresence-demo -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
  ```
4. Expose a sample Node.js application leveraging TSB
  ```
  cat <<EOF > tsb-telepresence-demo-gw-ingress.yaml
  ---
  apiVersion: gateway.tsb.tetrate.io/v2
  kind: IngressGateway
  metadata:
    organization: tetrate
    name: tsb-telepresence-demo-gw-ingress
    group:  tsb-telepresence-demo-gw
    workspace:  tsb-telepresence-demo-ws
    tenant:  tsb-telepresence-demo
  spec:
    workloadSelector:
      namespace: tsb-telepresence-demo
      labels:
        app: tsb-telepresence-demo-gw
    http:
      - name: verylargejavaservice
        hostname: verylargejavaservice.tsb-telepresence-demo.tetrate.io
        port: 8080
        routing:
          rules:
            - route:
                host: "tsb-telepresence-demo/verylargejavaservice.tsb-telepresence-demo.svc.cluster.local"
  EOF
  tctl apply -f tsb-telepresence-demo-gw-ingress.yaml
  ```
5. Validate exposed service
  ```
  export INGRESS_IP=`(kubectl get svc tsb-telepresence-demo-gw --output=jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")`
  curl -I http://$INGRESS_IP -H"Host: verylargejavaservice.tsb-telepresence-demo.tetrate.io"
  ```
  
  Expected output: 

  ```
  curl http://$INGRESS_IP -H"Host: verylargejavaservice.tsb-telepresence-demo.tetrate.io"  | grep color
  <h1 style="color:green">Welcome to the EdgyCorp WebApp</h1>
  ```

6. Setup up a local development environment, perform steps from 4 to 6: https://www.telepresence.io/docs/latest/quick-start/qs-node/

# Support materials

https://www.telepresence.io/docs/latest/reference/architecture/
https://www.telepresence.io/docs/latest/concepts/devloop/
https://www.telepresence.io/docs/latest/release-notes/
