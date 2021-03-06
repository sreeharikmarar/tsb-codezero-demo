# tsb-codezero-demo
Using CodeZero for local development with Tetrate Service Bridge


## Pre-requisite

* [CodeZero](https://docs.codezero.io/#/guides/installing) CLI installed on local
* [TSB Demo installation](https://docs.tetrate.io/service-bridge/1.4.x/en-us/setup/on_prem/demo-installation)
* TSB IngressGateway with LoadBalancer service

## Steps

### Application Deployment

Create a target namespace for the CodeZero demo, Enable istio-injection in the namespace.

```
$: kubectl create ns tsb-cz-demo
$: kubectl label namespace tsb-cz-demo istio-injection=enabled --overwrite
```

Deploy Blogger App

```
$: kubectl apply -f blogger_credentials.yaml -n tsb-cz-demo
$: kubectl apply -f blogger_migration_job.yaml -n tsb-cz-demo
$: kubectl apply -f blogger_deployment.yaml -n tsb-cz-demo
$: kubectl apply -f blogger_service.yaml -n tsb-cz-demo
```

### TSB Configuration

Create Tenant, Groups, Ingress, Application & API configuration 

Tenant

```
    apiVersion: api.tsb.tetrate.io/v2
    kind: Tenant
    metadata:
    organization: tetrate
    name: marketing
    spec:
    displayName: Marketing

$: tctl apply -f tsb/tenant.yaml
```

Workspace

```
    apiversion: api.tsb.tetrate.io/v2
    kind: Workspace
    metadata:
    organization: tetrate
    tenant: marketing
    name: marketing-ws
    spec:
    namespaceSelector:
        names:
        - "<cluster>/tsb-cz-demo"

$: tctl apply -f tsb/workspace.yaml
```

Groups

```
    apiVersion: gateway.tsb.tetrate.io/v2
    kind: Group
    metadata:
    organization: tetrate
    tenant: marketing
    workspace: marketing-ws
    name: blogger-gw
    spec:
    displayName: blogger-gateway
    namespaceSelector:
        names:
        - "<cluster>/tsb-cz-demo"
    configMode: BRIDGED
    ---
    apiVersion: traffic.tsb.tetrate.io/v2
    kind: Group
    Metadata:
    organization: tetrate
    tenant: marketing
    workspace: marketing-ws
    name: blogger-traffic
    spec:
    displayName: blogger-traffic-group
    namespaceSelector:
        names:
        - "<cluster>/tsb-cz-demo"
    configMode: BRIDGED
    ---
    apiVersion: security.tsb.tetrate.io/v2
    kind: Group
    Metadata:
    organization: tetrate
    tenant: marketing
    workspace: marketing-ws
    name: blogger-security
    spec:
    displayName: blogger-security-group
    namespaceSelector:
        names:
        - "<cluster>/tsb-cz-demo"
    configMode: BRIDGED

$: tctl apply -f tsb/groups.yaml
```

Ingress LoadBalancer

```
    apiVersion: install.tetrate.io/v1alpha1
    kind: IngressGateway
    metadata:
    name: tsb-gateway-blogger
    namespace: tsb-cz-demo
    spec:
    kubeSpec:
        service:
        type: LoadBalancer

$: kubectl apply -f tsb/ingress.yaml
```

Blogger Application

```
    apiversion: application.tsb.tetrate.io/v2
    kind: Application
    metadata:
    organization: tetrate
    tenant: marketing
    name: blogger
    spec:
    displayName: Blogger
    description: Blogger application
    workspace: organizations/tetrate/tenants/marketing/workspaces/marketing-ws
    gatewayGroup: organizations/tetrate/tenants/marketing/workspaces/marketing-ws/gatewaygroups/blogger-gw

$: tctl apply -f tsb/gateway.yaml
```

Blogger API

```
$: tctl apply -f tsb/api.yaml
```

### Verify Exposed Application in TSB

```
$: export INGRESS_IP=`(kubectl get svc tsb-gateway-blogger -n tsb-cz-demo --output=jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")`

$: curl -i http://$INGRESS_IP/api/v1/posts -H"Host: blogger.tetrate.com"
HTTP/1.1 200 OK
server: istio-envoy
date: Mon, 04 Apr 2022 17:27:52 GMT
content-type: application/json; charset=utf-8

{"posts":[{"id":1,"title":"How to build a ROR application","content":"Creating the Blog Application ??? This guide is designed for beginners who want to get started with creating a Rails application from scratch. It does","author":"Petr"}]}%
```


### Setup Local Development Environment for Blogger App

```
$: git clone git@github.com:sreeharikmarar/blogger.git
$: cd blogger
$: bundle instal
$: make config-copy
$: make docker.start
$: bundle exec rails s
```

### Verify Local application

```
$: curl http://localhost:3000/api/v1/posts

{"posts":[{"id":1,"title":"user1 - test post","content":"Recusandae minima consequatur. Expedita sequi blanditiis. Ut fuga et.","author":"user1"}]}
```

### Allow Rails Application to receieve requests from ngrok domain

Add the below line to `config/environments/development.rb` and restart the rails server

```
config.hosts << /.*\.ngrok\.io$/
```

## Intercept Blogger Traffic from remote cluster to local instance

```
$: czctl start

???  Starting daemon                                                                              Done
???  Start request received                                                                       Done
???  Initializing daemon version 1.4.0                                                            Done

$: czctl intercept service blogger -n tsb-cz-demo

???  Initiating intercept                                                                         Done
???  Intercept request received                                                                   Done
???  Validating intercept parameters                                                              Done
???  Starting intercept session for blogger in tsb-cz-demo                                        Done
???  Starting tunnel worker process                                                               Done
???  Updating routes for blogger in tsb-cz-demo                                                   Done
???  Adding intercept for blogger in tsb-cz-demo                                                  Done
???  Connecting remote service to local tunnel                                                    Done

```

### Verify traffic has been intercepted by CodeZero

Set `X-C6O-INTERCEPT: yes` header

```
$: curl -i  --location --request GET 'http://blogger.tetrate.com/api/v1/posts' \
--header 'X-C6O-INTERCEPT: yes' \
--data-raw ''
HTTP/1.1 200 OK
server: istio-envoy
date: Mon, 04 Apr 2022 17:37:20 GMT
content-type: application/json; charset=utf-8
ngrok-agent-ips: 2409:4073:4e1f:72ab:e035:8e0e:c93a:5553

{"posts":[{"id":1,"title":"user1 - test post","content":"Recusandae minima consequatur. Expedita sequi blanditiis. Ut fuga et.","author":"user1"}]}
```

You will see the response from the service running locally being intercepted. 

You can also open `ngrok` dashboard on http://localhost:4040 to inspect the requests. 


![001](images/001.png)


## Closing Intercept

```
$: czctl intercept service blogger -n tsb-cz-demo --clean

??? Initiating intercept                                             Done
??? Intercept request received                                       Done
??? Validating intercept parameters                                  Done
??? Closing intercept session for blogger in tsb-cz-demo             Done
??? Closing local tunnel worker process                              Done
??? Removing intercept for blogger in tsb-cz-demo                    Done
```