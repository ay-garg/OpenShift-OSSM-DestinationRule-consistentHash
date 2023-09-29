# OpenShift-OSSM-DestinationRule-consistentHash

Create `DestinationRule` with `consistentHash` load-balancing in OpenShift 4 with Service Mesh.

## Create a new project and add it to `SMMR`.
```
$ oc new-project consistenthash

$ oc edit smmr default -n istio-system
...
spec:
  members:
  - consistenthash
```

## Grant access to `anyuid` SCC for `default` service account in newly created project so the pods can come up. In the deployment YAML, the command and args are configured to create the `/var/www/html/index.html` file which requires root access and that's why pod is running as `0` user.
```
$ oc adm policy add-scc-to-user anyuid -z default
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"
```

## Deploy `httpd` sample app with sidecar injection enabled using the provided `httpd-deployment.yaml`. Deployment should be exposed to create SVC.
```
$ oc create -f httpd-deployment.yaml 
deployment.apps/httpd created

$ oc get pod
NAME                     READY   STATUS    RESTARTS   AGE
httpd-54498d7845-5k5tf   2/2     Running   0          10s
httpd-54498d7845-q68j2   2/2     Running   0          10s
httpd-54498d7845-ztpxc   2/2     Running   0          10s

$ oc expose deployment httpd 
service/httpd exposed
```

## Create a `virtualservice` with required host.
```
$ cat virtualservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpd-virtual-service
spec:
  hosts:
  - "*"
  gateways:
  - httpd-gateway 
  http:
    - route:
        - destination:
            host: httpd 
            port:
               number: 8080

$ oc create -f virtualservice.yaml
virtualservice.networking.istio.io/httpd-virtual-service created
```

## Create the destination rule with required consistentHash load-balancing trafficPolicy.
```
$ cat destinationrule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpd-destination-rule 
spec:
  host: httpd.consistenthash.svc.cluster.local 
  subsets:
  - labels:
      deployment: httpd
    name: httpd
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpCookie:
          name: APP_SESSION_ID 
          ttl: 900s

$ oc create -f destinationrule.yaml
destinationrule.networking.istio.io/httpd-destination-rule created
```

## Create the gateway in the application project.
```
$ cat gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpd-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "consistenthash.apps.ayush.example.com"

$ oc create -f gateway.yaml
gateway.networking.istio.io/httpd-gateway created
```

## The connections will be getting established to same pod even after multiple tries when route is accessed over web-browser.
```
$ oc get route -n istio-system | grep -i consistenthash
consistenthash-httpd-gateway-5cd5b839fbde900f   consistenthash.apps.ayush.example.com                             istio-ingressgateway   http2                                None
```
