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
  cat <<EOF > tsb-configuration.yaml
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
    workspace: tsb-telepresence-demo
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
    name: tsb-telepresence-demo
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
    workspace: tsb-telepresence-demo-traffic
    name: tsb-telepresence-demo
  spec:
    displayName: tsb-telepresence-demo-traffic
    namespaceSelector:
      names:
        - <cluster>/tsb-telepresence-demo
    configMode: BRIDGED
  ---
  apiVersion: security.tsb.tetrate.io/v2
  metadata:
    organization: tetrate
    tenant: tsb-telepresence-demo
    workspace: tsb-telepresence-demo-security
    name: tsb-telepresence-demo
  spec:
    displayName: tsb-telepresence-demo-security
    namespaceSelector:
      names:
        - <cluster>/tsb-telepresence-demo
    configMode: BRIDGED
  
  tctl apply -f tsb-configuration.yaml
  ```

2. Perform steps from 1 to 2: https://www.telepresence.io/docs/latest/quick-start/qs-node/

3. Install a sample Node.js application
  ```
  kubectl apply -n tsb-telepresence-demo -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
  ```
4. 