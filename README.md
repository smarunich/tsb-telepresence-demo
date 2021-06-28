# Using Telepresence with Tetrate Service Bridge

# Prerequisites

1. Telepresence binary
2. TSB control plane installed to cluster
3. kubectl, tctl, kubens
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

6. Patch application deployments 
  ``` 
  cat <<EOF > patch-telepresence-excludeOutboundPorts.yaml
  ---
  spec:
    template:
      metadata:
        annotations:
          traffic.sidecar.istio.io/excludeOutboundPorts: "6001,8081,8022"
  EOF
  kubectl patch deployment dataprocessingservice --type merge --patch "$(cat patch-telepresence-excludeOutboundPorts.yaml)"
  ```
  
7. Setup up a local development environment and lunch DataProcessingService locally, for more details refer to step 4 and 5 over: https://www.telepresence.io/docs/latest/quick-start/qs-node/
  
  Clone repository:

  ```
  git clone https://github.com/datawire/edgey-corp-nodejs.git
  Cloning into 'edgey-corp-nodejs'...
  remote: Enumerating objects: 441, done.
  ...
  ```

  Lunch DataProcessingService
  
  ```
  npm install && npm start
  ...
  Welcome to the DataProcessingService!
  { _: [] }
  Server running on port 3000
  ```
  Validate service 

  ```
  curl localhost:3000/color
  "blue"
  ```

8. Intercept DataProcessingService traffic

  ```
  telepresence connect
  kubens tsb-telepresence-demo
  telepresence intercept dataprocessingservice --port 3000
  Launching Telepresence Daemon v2.3.2 (api v3)
  Connecting to traffic manager...

  Using Deployment dataprocessingservice
  intercepted
      Intercept name    : dataprocessingservice
      State             : ACTIVE
      Workload kind     : Deployment
      Destination       : 127.0.0.1:3000
      Volume Mount Error: sshfs is not installed on your local machine
      Intercepting      : all TCP connections
  ```

  As dataprocessingservice will be processed locally, the color will change from green to blue:

  ```
  curl http://$INGRESS_IP -H"Host: verylargejavaservice.tsb-telepresence-demo.tetrate.io"  | grep color
  <h1 style="color:blue">Welcome to the EdgyCorp WebApp</h1>
  ```

  Stop the session and review site page again, the color will change to green again.

  ```
  telepresence leave dataprocessingservice
  curl http://$INGRESS_IP -H"Host: verylargejavaservice.tsb-telepresence-demo.tetrate.io"  | grep color
  <h1 style="color:green">Welcome to the EdgyCorp WebApp</h1>
  ```
# Support materials

https://www.telepresence.io/docs/latest/reference/architecture/

https://www.telepresence.io/docs/latest/concepts/devloop/

https://www.telepresence.io/docs/latest/release-notes/

https://www.telepresence.io/docs/latest/quick-start/qs-node/
