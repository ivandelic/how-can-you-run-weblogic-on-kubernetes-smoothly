# How to Setup WebLogic Standard Edition on Kubernetes With WebLogic Operator and Nginx

WebLogic plays a significant role in the cloud-native landscape nowadays, with the help of [WebLogic Kubernetes Operator](https://oracle.github.io/weblogic-kubernetes-operator/). How to set up WebLogic Domain in Kubernetes? How to containerize the domain? How to autoscale? Continue reading and explore highlights of WLS on k8s, together with a toolset that will make your ops manageable.

## Prerequisites
This guide assumes you have basic skills and knowledge about:
   - Docker CLI and containers (basic)
   - Oracle Cloud Infrastructure (basic)
   - Kubernetes and OKE (intermediate)
   - Image registry OCIR (basic)
   - WebLogic administration (intermediate)

Make sure you have:
1. Docker [installed](https://docs.docker.com/get-docker/) locally.
2. Kubernetes cluster provisioned and ready. I will use [OKE](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengoverview.htm) provisioned on OCI.
3. ```kubectl``` installed locally. You can follow the [installation docs](https://kubernetes.io/docs/tasks/tools/) and configure it by following [the guide](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm#localdownload).
4. Access to OCIR and the ability to push and pull container images.
5. Helm installed locally (package manager for K8s applications). It's required to install WebLogic Operator. Please follow the official [guide](https://helm.sh/docs/intro/install/).
6. Installed Nginx Ingress Controller. Please follow the [official instructions](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupingresscontroller.htm#settingupcontroller).

Familiarize yourself with [WebLogic Kubernetes Operator](https://oracle.github.io/weblogic-kubernetes-operator/) and proceed with Installation.

## Install WebLogic Domain and Operator

### 01 - Prepare Samples

1. Clone the Git repository with various examples locally:
    ```console
    git clone --branch v3.3.8 https://github.com/oracle/weblogic-kubernetes-operator
    ```
2. Visit [container-registry.oracle.com](https://container-registry.oracle.com/), log in with your Oracle Account, and accept terms on the right side of the screen.
   ![](image/conatiner-registry-accept-terms-1.png)
3. Login with Docker CLI on container-registry.oracle.com, using the same Oracle Account. You will need it to retrieve the base WebLogic image.
    ```console
    docker login container-registry-frankfurt.oracle.com
    ```
4. Pull WebLogic from the upper reposiotry locally, so the build preocess can use it.
    ```console
    docker pull container-registry-frankfurt.oracle.com/middleware/weblogic:14.1.1.0-11
    ```
5. Login with Docker CLI to OCIR. You will need OCIR to store final WebLogic image with the domain.
    ```console
    docker login eu-frankfurt-1.ocir.io
    ```

### 02 - Install WebLogic Operator
We will create a K8s namespace and deploy WebLogic Operator in it.

1. Create Kubernetes Namespace for WebLogic Kubernetes Operator:
    ```console
    kubectl create namespace edea-demo-weblogic-operator
    ```
2. Create Service Account for the Operator:
    ```console
    kubectl create serviceaccount -n edea-demo-weblogic-operator edea-demo-weblogic-operator-sa
    ```
3. Add WebLogic Kubernetes Operator charts repository to Helm:
    ```console
    helm repo add weblogic-operator https://oracle.github.io/weblogic-kubernetes-operator/charts --force-update
    ```
4. Use Helm, to install WebLogic Kubernetes Operator from the repository you have added in the step 3:
    ```console
    helm install demo-weblogic-operator weblogic-operator/weblogic-operator \
      --namespace edea-demo-weblogic-operator \
      --set image=ghcr.io/oracle/weblogic-kubernetes-operator:3.3.8 \
      --set serviceAccount=edea-demo-weblogic-operator-sa \
      --set "enableClusterRoleBinding=true" \
      --set "domainNamespaceSelectionStrategy=LabelSelector" \
      --set "domainNamespaceLabelSelector=weblogic-operator\=enabled" \
      --wait
    ```
5. You should get a confirmation from Helm like this:
    ```console
    NAME: demo-weblogic-operator
    LAST DEPLOYED: Thu Mar 31 13:24:30 2022
    NAMESPACE: edea-demo-weblogic-operator
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    ```
6. Check the installation by inspecting pods in the targeted namespace. All operator pods should be in a RUNNING state.
    ```console
    kubectl get pods -n edea-demo-weblogic-operator
    ```
7. If you inspect Helm installations, you will see it as installed in the specified namespace:
    ```console
    helm list -n edea-demo-weblogic-operator
    ```
   
### 03 Domain-Home-In-Image

#### 03.1 - Create WebLogic Domain (Domain-Home-In-Image)
After deploying WebLogic Operator, it's time to prepare demo domain and container images.

1. Create K8s namespace for WebLogic Domain:
    ```console
    kubectl create namespace edea-demo-weblogic-domain
    ```
2. Label freshly created namespace with ```weblogic-operator=enabled```
    ```console
    kubectl label ns edea-demo-weblogic-domain weblogic-operator=enabled
    ```
3. Upgrade ```demo-weblogic-operator``` with Helm, by providing Kubernetes Namespace to the Operator:
    ```console
    helm upgrade demo-weblogic-operator weblogic-operator/weblogic-operator \
      --namespace edea-demo-weblogic-operator \
      --reuse-values \
      --set "domainNamespaces={edea-demo-weblogic-domain}" \
      --wait
    ```
4. Position yourself in [freshly cloned](#clone-git-repo) git repository
    ```console
    cd weblogic-kubernetes-operator/
    ```
5. Create WebLogic credentials (e.g. ```weblogic/welcome1``` in domainUID ```edea-demo```):
    ```console
    kubernetes/samples/scripts/create-weblogic-domain-credentials/create-weblogic-credentials.sh -u weblogic -p welcome1 -n edea-demo-weblogic-domain -d edea-demo
    ```
6. Copy the template file (create-domain-inputs.yaml) for domain creation in the project folder. The template file contains the structure for domain creation that will be baked in the container image. This example uses Domain in Image mode to generate a container image with domain home embedded.
    ```console
    cp kubernetes/samples/scripts/create-weblogic-domain/domain-home-in-image/create-domain-inputs.yaml ../1-domain-home-in-image/edea-domain-inputs.yaml
    ```
7. Edit the file edea-domain-inputs.yaml and update properties with:
   * ```domainUID: edea-demo```
   * ```weblogicCredentialsSecretName: edea-demo-weblogic-credentials```
   * ```namespace: edea-demo-weblogic-domain```
   * ```domainHomeImageBase: container-registry-frankfurt.oracle.com/middleware/weblogic:14.1.1.0-11```
8. Remove ```clusterName: cluster-1``` form the edea-domain-inputs.yaml to make sure WebLogic cluster is not created. We need it to stay compliant with WebLogic Standard Edition licensing.
9. Run the next command to generate fresh container image with baked domain inside. Creation of domain is instructed by the template file from the previous step. The script will prepare new container image, based on the image provided from the above step.
    ```console
    kubernetes/samples/scripts/create-weblogic-domain/domain-home-in-image/create-domain.sh -i ../1-domain-home-in-image/edea-domain-inputs.yaml -o ../1-domain-home-in-image/edea-domain-output -u weblogic -p welcome1
    ```
10. You will se an output similar to this:
     ```console
     Successfully tagged domain-home-in-image:14.1.1.0-11
     [INFO   ] Build successful. Build time=87s. Image tag=domain-home-in-image:14.1.1.0-11
     @@ Info: Create domain edea-demo successfully.
     @@ Info: Domain edea-demo was created and will be started by the WebLogic Kubernetes Operator
     @@ Info: The following files were generated:
       edea-domain-output/weblogic-domains/edea-demo/create-domain-inputs.yaml
       edea-domain-output/weblogic-domains/edea-demo/domain.yaml
     @@ Info: Completed
     ```
11. Check the existence of a freshly generated container image with the domain inside:
    ```console
    docker images | grep domain-home-in-image
    ```
12. Tag the image with the proper OCIR data, samilarly to below:
    ```console
    docker tag domain-home-in-image:14.1.1.0-11 <region>.ocir.io/<namespace>/oracle/domain-home-in-image:14.1.1.0-11
    docker tag domain-home-in-image:14.1.1.0-11 eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/domain-home-in-image:14.1.1.0-11
    ```
13. Make sure you are logged in to OCIR:
    ```console
    docker login eu-frankfurt-1.ocir.io
    ```
14. Push the image to OCIR. You will need to make sure
    ```console
    docker push <region>.ocir.io/<namespace>/oracle/domain-home-in-image:14.1.1.0-11
    docker push eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/domain-home-in-image:14.1.1.0-11
    ```

#### 03.2 - Deploy WebLogic Domain to Kubernetes with Operator (Domain-Home-In-Image)
After you create a container image with an embedded WebLogic Domain, it's time to deploy the domain in the Kubernetes cluster with the Operator's help.
1. Edit file ```1-domain-home-in-image/edea-domain-output/weblogic-domains/edea-demo/domain.yaml``` and update it with following properties:
   * Change image with ```image: "eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/domain-home-in-image:14.1.1.0-11"```.
   * Add ```adminChannelPortForwardingEnabled: true``` under the ```adminServer``` section.
2. Apply freshly generated domain.yaml with kubectl:
    ```console
    cd ..
    kubectl apply -f 1-domain-home-in-image/edea-domain-output/weblogic-domains/edea-demo/domain.yaml
    ```
3. Wait for some time and verify domain contents with:
    ```console
    kubectl describe domain edea-demo -n edea-demo-weblogic-domain
    ```
4. Since we enabled ```adminChannelPortForwardingEnabled```, you can access the port-forwarded admin port to your local machine:
    ```console
   kubectl port-forward pods/edea-demo-admin-server -n edea-demo-weblogic-domain 7001:7001
   ```
5. Open your browser and go to ```http://localhost:7001/console```. You have set up credentials in step 5 of [Section 03](#03---create-weblogic-domain).

| :information_source: Note          |
|:-----------------------------------|
| If you need to refresh domain configuration based on ```domain.yaml``` file, you will need to run introspection. You can achieve it by adding or changing ```introspectVersion``` property. For example, you can type ```introspectVersion: "2"``` under the ```specs``` section. It will trigger updates of domain resources, including scaling and changes to channels. |

### 04 - Model-In-Image

#### 04.1 - Create WebLogic Domain (Model-In-Image)
1. Go to the folder
    ```console
    cd 2-model-in-image/model-images
    ```
2. Download weblogic-deploy.zip in the current folder:
    ```console
    curl -m 120 -fL https://github.com/oracle/weblogic-deploy-tooling/releases/latest/download/weblogic-deploy.zip -o ./weblogic-deploy.zip
    ```
3. Download imagetool.zip in the current folder and unzip it:
    ```console
    curl -m 120 -fL https://github.com/oracle/weblogic-image-tool/releases/latest/download/imagetool.zip -o ./imagetool.zip
    unzip imagetool.zip
    ```
4. Clear cache, if there is one previously generated:
   ```console
   ./imagetool/bin/imagetool.sh cache deleteEntry --key wdt_latest
   ```
5. Install WIT and reference WDT:
   ```console
   ./imagetool/bin/imagetool.sh cache addInstaller --type wdt --version latest --path ./weblogic-deploy.zip
   ```
6. Go in folder with WAR source:
   ```console
   cd ../archives/archive-v1/
   ```
7. Zip the archive:
   ```console
   zip -r ../../model-images/playground-model/archive.zip wlsdeploy
   ```
8. Go in the folder with model images:
   ```console
   cd ../../model-images
   ```
9. Build the image with inputs:
   ```console
    ./imagetool/bin/imagetool.sh update \
    --tag model-in-image:14.1.1.0-11 \
    --fromImage container-registry-frankfurt.oracle.com/middleware/weblogic:14.1.1.0-11 \
    --wdtModel      ./playground-model/playground.yaml \
    --wdtVariables  ./playground-model/playground.properties \
    --wdtArchive    ./playground-model/archive.zip \
    --wdtModelOnly \
    --wdtDomainType WLS \
    --chown oracle:root
   ```
10. You will se a confirmation:
   ```console
   [INFO   ] Build successful. Build time=32s. Image tag=model-in-image:14.1.1.0-11
   ```
11. Check the existence of a freshly generated container image with the domain inside:
    ```console
    docker images | grep model-in-image
    ```
12. Tag the image with the proper OCIR data, samilarly to below:
    ```console
    docker tag model-in-image:14.1.1.0-11 <region>.ocir.io/<namespace>/oracle/model-in-image:14.1.1.0-11
    docker tag model-in-image:14.1.1.0-11 eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/model-in-image:14.1.1.0-11
    ```
13. Make sure you are logged in to OCIR:
    ```console
    docker login eu-frankfurt-1.ocir.io
    ```
14. Push the image to OCIR. You will need to make sure
    ```console
    docker push <region>.ocir.io/<namespace>/oracle/model-in-image:14.1.1.0-11
    docker push eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/model-in-image:14.1.1.0-11
    ```

#### 04.2 - Deploy WebLogic Domain to Kubernetes with Operator (Model-In-Image)
1. Create secrets:
   ```console
   kubectl -n edea-demo-weblogic-domain create secret generic playground-weblogic-credentials --from-literal=username=weblogic --from-literal=password=welcome1
   kubectl -n edea-demo-weblogic-domain label secret playground-weblogic-credentials weblogic.domainUID=playground
   kubectl -n edea-demo-weblogic-domain create secret generic playground-runtime-encryption-secret --from-literal=password=my_runtime_password
   kubectl -n edea-demo-weblogic-domain label secret playground-runtime-encryption-secret weblogic.domainUID=playground
   ```
2. Edit file ```2-model-in-image/domain.yaml``` and update it with following properties:
   * Change image with ```image: "eu-frankfurt-1.ocir.io/frsxwtjslf35/oracle/model-in-image:14.1.1.0-11"```.
   * Add ```adminChannelPortForwardingEnabled: true``` under the ```adminServer``` section.
3. Apply freshly generated domain.yaml with kubectl:
    ```console
    cd ..
    kubectl apply -f 2-model-in-image/domain.yaml
    ```
4. Wait for some time and verify domain contents with:
    ```console
    kubectl describe domain playground -n edea-demo-weblogic-domain
    ```
5. Since we enabled ```adminChannelPortForwardingEnabled```, you can access the port-forwarded admin port to your local machine:
    ```console
   kubectl port-forward pods/edea-demo-admin-server -n edea-demo-weblogic-domain 7001:7001
   ```
6. Open your browser and go to ```http://localhost:7001/console```. You have set up credentials in step 5 of [Section 03](#03---create-weblogic-domain).

| :information_source: Note          |
|:-----------------------------------|
| If you need to refresh domain configuration based on ```domain.yaml``` file, you will need to run introspection. You can achieve it by adding or changing ```introspectVersion``` property. For example, you can type ```introspectVersion: "2"``` under the ```specs``` section. It will trigger updates of domain resources, including scaling and changes to channels. |

### 05 - Expose WebLogic Admin Server Through Ingress (Nginx)
WebLogic Operator created services accessible internally from the cluster. External users cannot still access the domain since it's not exposed through LoadBalancer or Ingress. Let's generate Ingress and expose the domain to the publicly available hostname.
1. Make sure edea-domain-ingress.yaml has the correct namespace and backend service name.
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: edea-demo-weblogic-ingress
     namespace: edea-demo-weblogic-domain
     annotations:
       kubernetes.io/ingress.class: "nginx"
   spec:
     rules:
       - http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: edea-demo-admin-server
                   port:
                     number: 7001
   ```
2. Apply the manifest:
   ```console
   kubectl apply -f edea-domain-ingress.yaml
   ```
3. Find the IP address of ```ingress-nginx-controller``` by checking ```EXTERNAL-IP``` column from the results:
   ```console
   kubectl get svc -n ingress-nginx
   ```
4. Open your browser and go to ```http://EXTERNAL-IP/console```.

## Uninstall WebLogic Domain and Operator (Optional)
1. ```console helm uninstall demo-weblogic-operator -n edea-demo-weblogic-operator```
2. ```console kubectl delete namespace edea-demo-weblogic-operator```
3. ```console kubernetes/samples/scripts/delete-domain/delete-weblogic-domain-resources.sh -d edea-wls-domain```
4. ```console kubectl delete edea-demo-weblogic-domain```
5. ```console docker rmi domain-home-in-image:14.1.1.0-11```
