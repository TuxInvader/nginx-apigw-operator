# NGINX Controller Pod Operator

This is an operator for managing the lifecycle of K8s deployment, enabling pod configuration by NGINX Controller. 

It is designed to work with the [docker-nginx-controller](https://github.com/nginxinc/docker-nginx-controller) container, so
you'll need to build yourself an NGINX Plus image from that project first.

The container referenced above will automatically register itself with NGINX Controller when it starts, but it's then up
to you to apply configuration to it. This project is an interrim measure until Controller fully supports Kubernetes. This
operator creates a new Custom Resource of `Gateway` under `apigw.nginx.com` group. You can then use the `Gateway` to define
a `Deployment` and link that deployments configuration to a Controller Gateway object using a matching `tag`

Once the Gateway has created the deployment, you can scale as normal, and changes to the pods will update the NGINX Controller.

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

The default operator will monitor all namespaces in kubernetes for `Gateway` resources in group `apigw.nginx.com`, you can limit this to
specific namespaces by providing a `WATCH_NAMESPACE` environment variable.

## Walk through

### Step One

Assuming you have the operator running on your k8's cluster, the first thing you need is a secrey to store the controller password.

```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: controller
data:
  user_password: cGFzc3dvcmQ=
```

### Step Two

You also need an NGINX Plus container which can automatically register itself with the NGINX Controller.
So build this one: [Docker NGINX Controller](https://github.com/nginxinc/docker-nginx-controller), and host it in a private repo.

You then need to deploy that container, via a `Gateway` resource.

```
apiVersion: apigw.nginx.com/v1alpha1
kind: Gateway
metadata:
  name: apigw
  namespace: apigw-team1
spec:
  instance_group: team1
  instance_location: k8s-team1
  user_email: "admin@nginx.com"
  fqdn: "10.0.0.4"
  secret: "controller"
  deployment:
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
            value: e282fe19f739fcbe160af6103b19b995
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

The Gateway resource will create a Kubernetes Deployment with the same name as the `Gateway`, and information from `deployment` as the spec.

You must provide the `Gateway` with the following information: `user_email`, `fqdn`, and `secret` are used to locate and login
to the NGINX Controller. `instance_location` is an instance location which should already exist on the controller, and should
match the `ENV_CONTROLLER_LOCATION` used by the container when it registers itself with controller. Finally `instance_group` is the 
"Instance Group" which should map to a matching tag on the Gateways defined on Controller.

The `tag` on the gateway in controller should be `ig:<name>`, so with a `instance_group` of "team1" in Kubernetes, the tag on NGINX Controller
should be `ig:team1`

The termination `preStop` and grace period are set so that the controller has time to remove configuration from them during
a scaling down event. The operator will also remove deleted pods from the controller instances list during such an event.

### Step Three

Once you have deployed the above `Gateway` you will want to go and create the `Controller Gateways` with the matching `tags` if you haven't
already. From this point on, the operator will update the Controller Gateway as and when the Gateway resource and its deployment/pods change.

## Notes

You will need a `Service` and some kind of `Ingress` as well as the deployment in order to get traffic into it. 

Controller doesn't know anything about the kubernetes service you want to manage with your NGINX ADC/API-GW, so in controller you
will likely need to use DNS name-resolution to resolve to the upstream services.


