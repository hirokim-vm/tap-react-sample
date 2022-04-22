# How to deploy Singe Page Applications with VMware Tanzu Application Platform (TAP)

## Create a new ClusterBuilder
Create a ClusterBuilder with the [kp CLI](https://github.com/vmware-tanzu/kpack-cli) ...
```
kp clusterbuilder create spa --buildpack tanzu-buildpacks/nodejs --buildpack tanzu-buildpacks/nginx --tag <your-container-registry-tag>
```
... or kubectl
```
cat << EOF | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: ClusterBuilder
metadata:
  name: spa
spec:
  order:
  - group:
    - id: tanzu-buildpacks/nodejs
    - id: tanzu-buildpacks/nginx
  serviceAccountRef:
    name: kp-default-repository-serviceaccount
    namespace: kpack
  stack:
    kind: ClusterStack
    name: default
  store:
    kind: ClusterStore
    name: default
  tag: <your-container-registry-tag>
EOF
```
The tag specifies the registry location where the builder will be created, therefore a Secret for the container registry credentials has to be available in the cluster.

## Create a Supplc Chain
In the current versions of TAP's **Out of the Box Supply Chain** it's not possible to override the default ClusterBuilder via **params** in the Workload configuration. 
Therefore we have to create a new Supply Chain for our new type of workload - Single Page Application. 

I used the **Out of the Box Supply Chain Basic** as a starting point for my customizations and just exported it from the cluster.
```
kubectl get clustersupplychain
kubectl get clustersupplychain <name-of-the-supplychain> -o yaml > cluster-supply-chain.yaml
```
If you have installed the **Out of the Box Supply Chain with Testing (and Scanning)** don't forget to adjust your Tekton Pipeline and be aware of the fact that most of the additional components of it don't support the **subPath** configuration I'm using here. 

After you have removed the status information, all annotations, labels etc., you have to change the **metadata.name**, configure our CluterBuilder for the **image-builder** step, ...
```
  - name: image-builder
    params:
    - name: serviceAccount
      value: default
    - name: clusterBuilder
      value: spa
```
... and finally set the selector at the end of the Supply Chain definition to make it selectable for our Workload.
```
  selector:
    apps.tanzu.vmware.com/workload-type: spa
```
An example is available [here](example-cluster-supply-chain.yaml) 

After that you can apply it to the cluster.
```
kubectl apply -f cluster-supply-chain.yaml
```
## SPA setup
It's required to have an [nginx.conf](frontend/nginx.conf) available in the root directory of your source code. Because of the subPath configuration in our Workload the root is `frontend/` The node version will be selected by the `engines.node` configuration in the packages.json.

## Apply the Workload
Let's have a closer look at the [workload.yaml](workload.yaml). 
The label `apps.tanzu.vmware.com/workload-type: spa` is required for the slection of our custom Supply Chain.

The `BP_NODE_RUN_SCRIPTS` environment variable is set to specify a script to run during build phase.

Let's now apply our workload.yaml. Because the `subPath` configuration is not supported by the tanzu CLI yet and removes it if we apply our workload-yaml with `tanzu apps workload create -f workload.yaml`, let's use kubectl to deploy it.
```
kubectl apply -f workload.yaml -n <your-developer-namespace>
```