# JupyterHub on Kubernetes


### We will install and test JupyterHub in k8s as a service from VK Cloud Solutions
#### Tested with Kubernetes 1.23.13, Ubuntu 22.04, Helm chart for JupyterHub 1.1.3, JupyterHub 1.4.2


## Running JupyterHub on Kubernetes useful links

### Official doc
https://zero-to-jupyterhub.readthedocs.io/en/latest/index.html   
https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/installation.html   



## Prerequisites

### (Optional) Create host VM  
It would be easier to create a host VM in cloud  
All work in the cloud can be done from that VM  
You can install kubectl, Helm, Docker and all other things on this VM and don`t mess with your own local machine  

How to create VM: https://mcs.mail.ru/help/ru_RU/create-vm/vm-quick-create  
How to connect: https://mcs.mail.ru/docs/ru/base/iaas/vm-start/vm-connect/vm-connect-nix 
Steps:  
1. Create VM  
2. Connect to VM with SSH  
3. Perform all steps described further in this instruction from this VM  
4. Enjoy cloud:)  


### Create K8s cluster in mcs and download kubeconfig
Instruction: https://mcs.mail.ru/help/ru_RU/k8s-start/create-k8s  
Kubernetes as a Service: https://mcs.mail.ru/app/services/containers/add/  

You may have trouble with Gatekeeper. So please delete it. 
https://mcs.mail.ru/docs/base/k8s/k8s-addons/k8s-gatekeeper/k8s-opa#udalenie

You have to install keystone-auth for k8s version 1.23 or higer
More information about changes see by link 
https://mcs.mail.ru/docs/base/k8s/concepts/access-management#1509-7

To install use instruction https://mcs.mail.ru/docs/base/k8s/connect/kubectl#9980-5
Don't forget to run after installation keystone-auth
```console
source /home/ubuntu/.bashrc
```

### Install kubectl
[https://mcs.mail.ru/help/ru_RU/k8s-start/connect-k8s  ](https://mcs.mail.ru/docs/ru/base/k8s/connect/kubectl)  
https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/  

```console
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Set path to kubeconfig for kubectl
```console
export KUBECONFIG=/replace_with_path/to_your_kubeconfig.yaml
```
Replace credentials in your_kubeconfig.yaml
```console
   - name: "OS_PASSWORD"
     value: "vkcloud_account_password"
```


### also it will be easier to work with kubectl while enabling autocomplete and using alias
```console
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
```

### Install helm
https://helm.sh/docs/intro/install/
```console
curl https://raw.githubusercontent.com/helm/helm/HEAD/scripts/get-helm-3 | bash
```

### (Optional) Install Docker if you want to build your own images for JupyterHub and log into a Docker registry  
https://docs.docker.com/engine/install/ubuntu/  
https://docs.docker.com/engine/reference/commandline/login/  
https://ropenscilabs.github.io/r-docker-tutorial/04-Dockerhub.html   

### (Mantadory) Repeat steps blow
Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```console 
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
Add Docker’s official GPG key:
```console
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
Use the following command to set up the repository:
```console
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


## JupyterHub installation and testing part

### Install JupyterHub
```console
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/ --insecure-skip-tls-verify
helm repo update
```

Also we need to mark one of storage classes as default for successful installation.  
ATTENTION: WATCH your k8s cluster availability zone. Storage class must be equal k8s cluster zone.
If cluster in GZ1 or any other zone, you should patch storageclass with this zone.
```console
kubectl get storageclass
kubectl patch storageclass csi-ceph-ssd-ms1-retain  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```


#### Create config for basic installation
warning: this config for demo use only! NOT A PRODUCTION SOLUTION
```console
nano config_basic.yaml
#paste this to config_basic.yaml

#Change storageClass for your availability zone. For example: storageClass: csi-ceph-ssd-gz1-retain
singleuser:
  defaultUrl: "/lab"
  storage:
    dynamic:
      storageClass: csi-ceph-ssd-ms1-retain
  cpu:
    limit: .5
    guarantee: .5
  memory:
    limit: .256
    guarantee: .512
hub:
  config:
    Authenticator:
      admin_users:
        - admin
      allowed_users:
        - your_another_non_admin_user
#DummyAuthenticator not for production
    DummyAuthenticator:
      password: insertyourpasswordhereMVeP2VXfr
    JupyterHub:
      authenticator_class: dummy
``` 

#### Apply basic config and install JupyterHub
```console
helm upgrade --cleanup-on-fail \
  --install defaultinstall jupyterhub/jupyterhub --insecure-skip-tls-verify \
  --namespace jupyterhub \
  --create-namespace \
  --version=1.1.3 \
  --values config_basic.yaml \
  --timeout 20m0s
```

To access JupyterHub we need to find external ip  
```console
kubectl get services -n jupyterhub
```
Look for LoadBalancer Service type. Then look for external ip.  
You can access JupyterHub by entering this external ip to browser.  


For debug and troubleshouting
```console
kubectl get pods -n jupyterhub
kubectl get events -n jupyterhub
kubectl describe pod <pod-name> -n jupyterhub
kubectl logs <POD_NAME> -n jupyterhub
```

#### Create config for advanced installation
Let`s add some security measures


```console
nano config_lb.yaml
#paste this to config_lb.yaml

singleuser:
  defaultUrl: "/lab"
  storage:
    dynamic:
#you could use different storage classes
#get storage classes with: kubectl get storageclasses.storage.k8s.io 
      storageClass: csi-ceph-ssd-ms1-retain 
  cpu:
    limit: .5
    guarantee: .5
  memory:
    limit: .256
    guarantee: .512
hub:
  config:
    Authenticator:
      admin_users:
        - admin
      allowed_users:
        - your_another_non_admin_user
#DummyAuthenticator not for production
    DummyAuthenticator:
      password: insertyourpasswordhereMVeP2VXfr
    JupyterHub:
      authenticator_class: dummy
proxy:
  service:
#if you set loadBalancerSourceRanges, you can access JupyterHub only from ip address from this setting.
#you can set a bunch of IP adresses
#https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/security.html#restricting-load-balancer-access
    loadBalancerSourceRanges:
      - PLACE_YOUR_IP_HERE
      - PLACE_ANOTHER_YOUR_IP_HERE_OR_REMOVE_THIS_LINE
#EXAMPLE
#      - 91.74.148.161/32
``` 


#### Apply new config and upgrade JupyterHub
```console
helm upgrade --cleanup-on-fail \
  --install defaultinstall jupyterhub/jupyterhub --insecure-skip-tls-verify \
  --namespace jupyterhub \
  --version=1.1.3 \
  --values config_lb.yaml \
  --timeout 20m0s
```
You can check versions of Helm chart and JupyterHub here:   
https://jupyterhub.github.io/helm-chart/   


Also we can enable https and integrate JupyterHub with Github for authentication  
Read more here:    
https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/security.html#https   
https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/authentication.html#github    
https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch     

You can find working examples of config.yaml in config directory in this repo.   
Read config_https_github.yaml  




## INTEL oneAPI demo 

You could read more about oneAPI:    
https://software.intel.com/content/www/us/en/develop/tools/oneapi/ai-analytics-toolkit.html  
https://medium.com/intel-analytics-software/save-time-and-money-with-intel-extension-for-scikit-learn-33627425ae4   

You need to install Docker if you want to build your own image or you could use our image: mcscloud/jupyter-ds-intel-mcs:v2  
#### Login to Docker Hub  
```console
sudo docker login --username=YOUR_DOCKERHUB_USER_NAME
#sudo docker login --username=mcscloud
```
additional instruction about Docker Hub   
https://jsta.github.io/r-docker-tutorial/04-Dockerhub.html


#### Create Dockerfile 
```console
#make separate dir
mkdir ~/intel_based_docker_image && cd ~/intel_based_docker_image

#then create Dockerfile
nano Dockerfile

#paste this to Dockerfile
FROM jupyter/datascience-notebook:hub-1.4.2
RUN pip install --no-cache-dir nbgitpuller
#Доп информация про nbgitpuller
#https://github.com/jupyterhub/nbgitpuller 
#Install git extension
RUN pip install --no-cache-dir jupyterlab-git
#Install Intel part
RUN conda install -c conda-forge scikit-learn-intelex
```

#### Build and push image
Let`s build custom image with Intel ML package
```console
export YOUR_DOCKER_REPO=
#example export YOUR_DOCKER_REPO=mcscloud

sudo docker build -t jupyter-ds-intel-mcs .

sudo docker images
#find image id and copy it

sudo docker tag YOUR_IMAGE_ID  $YOUR_DOCKER_REPO/jupyter-ds-intel-mcs:v2
sudo docker push $YOUR_DOCKER_REPO/jupyter-ds-intel-mcs:v2
```

#### Create config for JupyterHub with Intel image
If you want to test Intel libraries for ML you need more resources  
As you can see we add cpu and memory requirements to config under singleuser part  

```console
nano config_intel.yaml
#paste this to config_intel.yaml

singleuser:
  defaultUrl: "/lab"
  storage:
    dynamic:
#you could use different storage classes
#get storage classes with: kubectl get storageclasses.storage.k8s.io 
      storageClass: csi-ceph-ssd-ms1-retain
  cpu:
    limit: 3
    guarantee: 2
  memory:
    limit: 3G
    guarantee: 512M
  # Defines the default image
  image:
    name: jupyter/minimal-notebook
    tag: hub-1.4.2
  profileList:
    - display_name: "Minimal environment"
      description: "To avoid too much bells and whistles: Python."
      default: true
    - display_name: "Tensorflow"
      description: "If you want the additional bells and whistles: Python, R, and Julia."
      kubespawner_override:
        image: jupyter/tensorflow-notebook:hub-1.4.2
    - display_name: "Spark environment"
      description: "The Jupyter Stacks spark image!"
      kubespawner_override:
        image: jupyter/all-spark-notebook:hub-1.4.2
    - display_name: "JupyterLab with Intel libraries"
      description: "Use some Intel optimizations"
      kubespawner_override:
        image: PLACE_YOUR_DOCKER_REPO_OR_USE_mcscloud/jupyter-ds-intel-mcs:v2
#       image: mcscloud/jupyter-ds-intel-mcs:v2
hub:
  config:
    Authenticator:
      admin_users:
        - admin
      allowed_users:
        - your_another_non_admin_user
#DummyAuthenticator not for production
    DummyAuthenticator:
      password: insertyourpasswordhereMVeP2VXfr
    JupyterHub:
      authenticator_class: dummy
proxy:
  service:
#if you set loadBalancerSourceRanges, you can access JupyterHub only from ip address from this setting.
#you can set a bunch of IP adresses
#https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/security.html#restricting-load-balancer-access
    loadBalancerSourceRanges:
      - PLACE_YOUR_IP_HERE
      - PLACE_ANOTHER_YOUR_IP_HERE_OR_REMOVE_THIS_LINE
#EXAMPLE
#      - 91.74.148.161/32
``` 

#### Apply new config and upgrade JupyterHub
```console
helm upgrade --cleanup-on-fail \
  --install defaultinstall jupyterhub/jupyterhub --insecure-skip-tls-verify \ \
  --namespace jupyterhub \
  --version=1.1.3 \
  --values config_intel.yaml \
  --timeout 20m0s
```

For testing Intel library clone repo https://github.com/intel/scikit-learn-intelex with Git extension already installed in Jupyter  
You can find Git extension on the left vertical bar  
After cloning repo you can find test Notebooks in scikit-learn-intelex/examples/notebooks/

To use JupyterHub, enter the external IP for the proxy-public service in to a browser.  
```console
kubectl get service -n jupyterhub
```

#### Help
If you have any questions you can ask me here:  
Telegram @volinski  
Email a.n.volinski@ya.ru  
