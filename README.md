# NGINX Controller Pod Operator

This is an operator for managing the lifecycle of K8s pods which are configured by NGINX Controller. 

It is designed to work with the [docker-nginx-controller](https://github.com/nginxinc/docker-nginx-controller) container, so
you'll need to build yourself an NGINX Plus image from that project first.

The container referenced above will automatically register itself with NGINX Controller when it starts, but it's then up
to you to apply configuration to it. This project is an interrim measure until Controller fully supports Kubernetes, the
operator will monitor deployments with a `nginx-apigw-instance-group` label, and then search the `Gateways` on Controller
for a match. The pods are then added to any matching `Gateways` so that Controller can then configure them.

## Building and Deploying

### Option 1: Build with Operator SDK

1. Follow the Operator SDK Install structions here [https://sdk.operatorframework.io/docs/installation/](https://sdk.operatorframework.io/docs/installation/)
2. Clone this repository
3. build `make docker-build docker-push IMG=<some-container-registry>/igmonitor:latest`
4. make deploy

If you used a private container registry then you will need to ensure that your deployment can authenticate. See this doc
for more instructions on that [bundles and private registries](https://sdk.operatorframework.io/docs/olm-integration/cli-overview/#private-bundle-and-catalog-image-registries)

### Option 2: Use my image

If you don't have the SDK, then you can always grab my image from Docker hub: `docker.io/tuxinvader/igmonitor:latest` and deploy it
by using/tweaking the manifests in the `examples/cluster` folder.

### Monitoring

The default deployment parameters will have created an `apigw-operator-system` namespace and a `apigw-operator-controller-manager`
pod which is now watching for resources. You can see its logs with:

```
kubectl -n apigw-operator-system logs deployment.apps/apigw-operator-controller-manager manager  -f
```

## Usage

The default operator will monitor all namespaces in kubernetes for `Controller` resources in group `apigw.nginx.com`, you can limit this to
specific namespaces by providing a `WATCH_NAMESPACE` environment variable.

The operator also monitors deployments which are labelled with `nginx-apigw-instance-group`. This label should match a tag on the `Gateway` objects
on your NGINX Controller. The operator will attempt to link any gateways tagged with a matching value to the pods in the labelled deployment.

## Walk through

### Step One

Assuming you have the operator running on your k8's cluster, the first thing you need is a controller resource and a kubernetes secret.

The secret

```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: controller
data:
  user_password: cGFzc3dvcmQ=
```

The Controller

```
apiVersion: apigw.nginx.com/v1alpha1
kind: Controller
metadata:
  name: controller1
  namespace: apigw-team1
spec:
  user_email: "admin@nginx.com"
  secret: "controller"
  fqdn: "10.0.0.4"
  validate_certs: false
```

### Step Two

You also need an NGINX Plus container which can automatically register itself with the NGINX Controller.
So build this one: [Docker NGINX Controller](https://github.com/nginxinc/docker-nginx-controller), and host it in a private repo.

You then need to deploy that container, using a manifest something like this:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apigw-dep
  namespace: apigw-team1
  labels:
    nginx-apigw-instance-group: team1
    nginx-apigw-instance-location: k8s-team1
    nginx-apigw-controller: controller1
spec:
  selector:
    matchLabels:
      app: apigw-sp
  replicas: 2
  template:
    metadata:
      labels:
        app: apigw-sp
    spec:
      terminationGracePeriodSeconds: 30
      imagePullSecrets:
        - name: f5reg
      containers:
      - image: f5reg.azurecr.io/nginx/nginx-agent:latest
        imagePullPolicy: "Always"
        name: nplus-apigw
        ports:
        - containerPort: 80
        env:
        - name: ENV_CONTROLLER_API_KEY
          value: eeeeeefffffffcbe160af6103b19b995
        - name: ENV_CONTROLLER_LOCATION
          value: k8s-team1
        - name: ENV_CONTROLLER_INSTANCE_GROUP
          value: team1
        lifecycle:
          preStop:
            exec:
              command: [
                # Allow Controller to remove us from services before shutting down
                "/bin/sleep", "25"
              ]
```

There are a number of labels in the metadata: `nginx-apigw-controller` names the `Controller` resource in the same namespace
which contains the location and authentication information for the controller. `nginx-apigw-instance-location` is an instance
location which should already exist on the controller, and should match the `ENV_CONTROLLER_LOCATION` used by the container
when it registers itself with controller. Finally `nginx-apigw-instance-group` is the "Instance Group" which should map to
a matching tag on all `gateways` which you want this deployment to host. 

The `tag` on the gateway should be `ig:<name>`, so with a `nginx-apigw-instance-group` of "team1", the tag should be `ig:team1`

The termination `preStop` and grace period are set so that the controller has time to remove configuration from them during
a scaling down event. The operator will also remove deleted pods from the controller instances list during such an event.

### Step Three

Once you have deployed the above `Deployment` you will want to go and create the `Gateways` with the matching `tags` if you haven't
already. From this point on, the operator will update the gateway as and when the deployment pods change.

## Notes

You will need a `Service` and some kind of `Ingress` as well as the deployment in order to get traffic into it. 

Controller doesn't know anything about the kubernetes service you want to manage with your NGINX ADC/API-GW, so in controller you
will likely need to use DNS name-resolution to resolve to the upstream services.


