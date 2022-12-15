# OCCD Kuma Hands-On-Learning Session

## Preparation

The following software needs to be installed:
- `git`
- `docker`
- `kubectl`
- `helm`
- `kubens/kubectx`, `httpie`, `yq`, `jq` (*all optional*)
- `kumactl` (for observability)

**Important**: Your router must allow DNS rebind for the url `nip.io`.
- To setup this in your FRITZ!Box go to `Heimnetz > Netzwerk > Netzwerkeinstellungen` and search for `DNS-Rebind-Schutz` on the bottom of the page

- For Windows users please use `WSL/WSL2`.


## Training session

### Step 1: Install k3d cluster

```sh
git clone https://github.com/FabianHardt/k3d-bootstrap-cluster
cd k3d-bootstrap-cluster
sudo ./create-sample.sh
```
Don't deploy the HttpBin service. The setup will ask for a few items, just press enter but on `Deploy httpbin sample? (Yes/No) [Yes]:` type `No`
The nginx Ingress is important for this session!

Keep track of the installation if ou want to:
```
watch kubectl get pod,deployment,service,ingress -A --field-selector=metadata.namespace!=kube-system
```


### Step 2: Install Kuma

```sh
git clone https://github.com/d4kine/occd-kuma-hol
cd occd-kuma-hol
./install/install-kuma.sh
```

After deployment, the Kuma GUI is exposed via ingress via http://kuma.127-0-0-1.nip.io:8080 (or http://localhost:5681/gui/#/ with port-forward)


### Step 3: Deploy demo app

```sh
kubectl apply -f demo/
```
Verify the deployment by calling http://frontend.127-0-0-1.nip.io:8080/ (or http://localhost:5000 with port-forward)


### Step 4: Configure TrafficRoutes

*Scenario: 1 frontend, 3 backends (v0,v1,v2), 1 redis, 1 postgres*
- Traffic will be routed 80% to v0, 20% to v1 & 0% to v2
- It's only possible to call v2 with the header-atribute: `version: v2`

```sh
kubectl apply -f traffic-routes/
```


### Step 5: Configure mTLS

#### Add Ingress to Mesh

For mTLS it's necessary, that the ingress is included in the mesh. To accomplish that, we will patch the deployment to add a sidecar and tag it as a gateway:

```sh
kubectl label namespace ingress-nginx kuma.io/sidecar-injection=enabled
kubectl -n ingress-nginx patch deployments/ingress-nginx-controller -p '{"spec":{"template":{"metadata":{"annotations":{"kuma.io/gateway":"enabled"}}}}}'
kubectl -n ingress-nginx delete pod --all
```

Verify with:
```sh
kubectl -n ingress-nginx get deployments/ingress-nginx-controller -o yaml | yq .spec.template.metadata
```

The Ingress-controller should also be present as service in the kuma dashboard named `ingress-nginx-controller_ingress-nginx_svc_80`

#### Send request into mesh

To test the inactive tls-config, we send a request from another namespace to a mesh-included-service:
```sh
kubectl -n outside-mesh exec -it $(kubectl -n outside-mesh get pods --no-headers -o custom-columns=":metadata.name") -- sh

curl backend.kuma-demo.svc.cluster.local:3001
```

#### Activate mTLS

```sh
kubectl apply -f mtls/
```

#### Send request into mesh

From outside:
```sh
# from outside
kubectl -n outside-mesh exec -it $(kubectl -n outside-mesh get pods --no-headers -o custom-columns=":metadata.name") -- sh
curl backend.kuma-demo.svc.cluster.local:3001
```

Within the mesh:
```sh
kubectl -n kuma-demo exec -it $(kubectl -n kuma-demo get pods --no-headers -o custom-columns=":metadata.name" | grep "demo-app-") -c kuma-fe -- sh
curl backend:3001 -vI
```


### Step 6: Configure TrafficPermissions

- **Requires mTLS and Traffic Permissions are an inbound policies**

*Scenario*: Connections are only allowed from `Frontend > Backend > Postgres` but not `Backend > Redis`

```sh
kubectl delete trafficpermissions --all
kubectl apply -f traffic-permissions/
```


### Troubleshooting

In case, that the ingress does not work (maybe dns rebound or config), you can execute the attached script `./port-forward.sh` and use the following URLs:
- For Kuma GUI: http://localhost:5681/gui/#/
- For demo frontend: http://localhost:5000
