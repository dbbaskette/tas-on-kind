# tas-on-kind


Preqrequisites | MAC Install CMD 
---------|----------
homebrew | ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
kubectl | brew install kubernetes-cli
docker |  Download from docker.com and Install dmg
kind | brew install kind
BOSH cli | brew install cloudfoundry/tap/bosh-cli
kapp, kbld, ytt | brew install ytt kbld kapp
cf cli | brew install cloudfoundry/tap/cf-cli
k9s (not really required, but helpful) | brew install derailed/k9s/k9s  
TAS for K8s | https://network.pivotal.io/products/tas-for-kubernetes


## Installing TAS for K8s on KIND
1. Verify Docker has at least 8GB of RAM
1. Create work directory and cd into it.
    ```
    mkdir tasdesktop; cd taskdesktop
    ```
1. Download a KIND Cluster Configuration file
    ```
    curl https://raw.githubusercontent.com/cloudfoundry/cf-for-k8s/master/deploy/kind/cluster.yml > cluster.yml
    ```
1. Create your KIND cluster
    ```
    kind create cluster --name tasdesktop --config=cluster.yml --image kindest/node:v1.16.4
    ```
1. Download the tas4k8s tarball and place it in into tasdesktop directory and untar it
1. Download remove-resource-requirements config file
    ```
    curl https://raw.githubusercontent.com/cloudfoundry/cf-for-k8s/master/config-optional/remove-resource-requirements.yml > custom-overlays/remove-resource-requirements.yml
    ```
1. Create Nodeport custom-overlay
    ```
    cat > custom-overlays/use-nodeport-for-ingress.yml <<EOF
    #@ load("@ytt:overlay", "overlay")

    #@overlay/match by=overlay.subset({"kind":"Service","metadata":{"name":"istio-ingressgateway"}})
    ---
    spec:
    #@overlay/match
    type: NodePort

    ports:
    #@overlay/match by="name"
    - name: http2
        #@overlay/match missing_ok=True
        nodePort: 31080
    #@overlay/match by="name"
    - name: https
        #@overlay/match missing_ok=True
        nodePort: 31443
    EOF
    ```
1. Move load-balancer config file out of active configuration
    ```
    mv custom-overlays/replace-loadbalancer-with-clusterip.yaml config-optional/.
    ```
1. Create a config-values directory
    ```
    mkdir config-values
    ```
1. Create the System Registry file and then fill it in with appropriate values
    ```
    cat > config-values/system-registry-values.yml <<EOF
    #@data/values
    ---
    system_registry:
      hostname: registry.pivotal.io
      username: "YOUR-TANZU-NETWORK-REGISTRY-USERNAME"
      password: "YOUR-TANZU-NETWORK-REGISTRY-PASSWORD"
    EOF
1. Create the App Registry file and then fill it in with appropriate values
    ```
    cat > config-values/app-registry-values.yml <<EOF
    #@data/values
    ---
    app_registry:
        hostname: "https://index.docker.io/v1/"
        repository: "YOUR_DOCKERHUB_USERNAME"
        username: "YOUR_DOCKERHUB_USERNAME"
        password: "YOUR DOCKERHUB PW"
    EOF
1. Generate defaults for the install 
    ```
     bin/generate-values.sh -d "vcap.me" > config-values/deployment-values.yml
    ```
1. Install the install
    ```
    ./bin/install-tas.sh ./config-values
    ```




