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
jq | brew install jq
yq | brew install yq
helm | brew install helm
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
1. Download use-nodeport-for-ingress.yml
    ```
    curl https://raw.githubusercontent.com/cloudfoundry/cf-for-k8s/ed4c9ea79025bb4767543cb013d3c854d1cd2b72/config-optional/use-nodeport-for-ingress.yml > custom-overlays/use-nodeport-for-ingress.yml
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
1. Install TAS on K8S
    ```
    ./bin/install-tas.sh ./config-values
    ```
## Test TAS for Kubernetes Install
1. Login to the API Server
    ```
    cf api api.vcap.me --skip-ssl-validation
    cf auth admin "$(bosh interpolate config-values/deployment-values.yml --path /cf_admin_password)"
    ```
1. Enable diego_docker feature flag (temp BETA requirement)
    ```
    cf enable-feature-flag diego_docker
    ```
1. Create Org and Space for test
    ```
    cf create-org test-org
    cf create-space -o test-org test-space
    cf target -o test-org -s test-space
    ```
1. Push Test App
    ```
    cf push demo-app -o rseroter/simple-k8s-app-kpack
    ```
## Minibroker Install
1. Create Namespace for minibroker and Install with helm
    ```
    kubectl create ns minibroker
    helm repo add minibroker https://minibroker.blob.core.windows.net/charts
    helm  repo update
    helm install  minibroker --namespace minibroker minibroker/minibroker --set "deployServiceCatalog=false" --set "defaultNamespace=minibroker"
    ```
1. Create Service Broker in TAS for K8s and enable access to the services
    ```
    cf create-service-broker minibroker user pass http://minibroker-minibroker.minibroker.svc.cluster.local
    cf service-access
    cf enable-service-access mysql
    cf enable-service-access redis
    cf enable-service-access mongodb
    cf enable-service-access mariadb
    ```
1. Verify services are available in the Marketplace
    ```
    cf marketplace
    ```
1. Create a mariadb service instance
    ```
    cf create-service mariadb 10-3-22 mariadb-svc -c '{"db": {"name": "my_database"}}'
    ```
1. Monitor the creation until it reports _**create succeeeded**_
    ```
    cf service
    ```

## Deploying the Demo Application
1. Clone the Demo App Repo and then go into the repo sirectory
    ```
    git clone https://github.com/dbbaskette/todos-pcf.git
    cd todos-pcf
    ```
1.  Push todos-ui to TAS4K8s. This is an example of pushing a prebuilt Docker container from a Repo. The Dockerfile is included if you would like to build your own version and push it to your own Docker Repo. Just remember to change the manifest.yml.
    ```
    cd todo-ui
    cf push 
    ```
1. Push todo-service to TAS4K8s. This is an example of pushing source code to TAS4K8s. The embedded Tanzu Build Service will build the source code and then package it in a container with all its dependencies.
    ```
    cd ../todo-service
    cf push
    ```
1. After the push succeeds, browse to http://todo.vcap.me





## Restarting TAS Cluster
If you want to shutdown the KIND docker container, or it shuts down for any reason, the KIND cluster will not restart automatically. Create a restart.sh and run it to restart your KIND Cluster.

1. Install yq (brew install yq)
1. Create restart-kind.sh and make it executable
    ``` 
    #!/usr/bin/env bash
    KIND_CLUSTER="tasdesktop"
    KIND_CTX="kind-${KIND_CLUSTER}"

    for container in $(kind get nodes --name ${KIND_CLUSTER}); do
        [[ $(docker inspect -f '{{.State.Running}}' $container) == "true" ]] || docker start $container
    done
    sleep 1
    docker exec ${KIND_CLUSTER}-control-plane sh -c 'mount -o remount,ro /sys; kill -USR1 1'
    kubectl config set clusters.${KIND_CTX}.server $(kind get kubeconfig --name ${KIND_CLUSTER} -q | yq read -j - | jq -r '.clusters[].cluster.server')
    kubectl config set clusters.${KIND_CTX}.certificate-authority-data $(kind get kubeconfig --name ${KIND_CLUSTER} -q | yq read -j - | jq -r '.clusters[].cluster."certificate-authority-data"')
    kubectl config set users.${KIND_CTX}.client-certificate-data $(kind get kubeconfig --name ${KIND_CLUSTER} -q | yq read -j - | jq -r '.users[].user."client-certificate-data"')
    kubectl config set users.${KIND_CTX}.client-key-data $(kind get kubeconfig --name ${KIND_CLUSTER} -q | yq read -j - | jq -r '.users[].user."client-key-data"')
    ```  


