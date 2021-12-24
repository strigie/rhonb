## OCP IPI on AWS with Calico eBPF CNI

### Introduction
This document will provide an HOWTO install Openshift on AWS with Calico eBPF CNI for lab purposes.

### Prerequisites
* an AWS account with a public route53 DNS zone
* a Red Hat account
* a computer to run the installation from. This was created on Fedora 35 x86_64

### Setup software
* Install pip and git
	
	```bash
	sudo dnf -y install pip git
	```
* Create a workspace directory and clone this repository

	```bash
	mkdir dev && cd dev
	git clone https://github.com/strigie/rhonb
	```
* Install ansible and required python modules

	```bash
	pip install -r rhonb/route53/requirements.txt
	```
	
* Install AWS CLI tool

	```bash
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
unzip /tmp/awscliv2.zip
sudo /tmp/aws/install
	```
	Refer to https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

* Install the Openshift installer and oc command line interface
	
	```bash
	curl "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.9.12/openshift-install-linux-4.9.12.tar.gz" -o "/tmp/openshift-install.tar.gz"
	tar xvzf /tmp/openshift-install.tar.gz openshift-install -C ~/.local/bin/
	```
	```bash
	curl "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.9.12/openshift-client-linux-4.9.12.tar.gz" -o "/tmp/openshift-client.tar.gz"
	tar xvzf /tmp/openshift-client.tar.gz oc -C ~/.local/bin/
	```
	Note: You can open the browse directories in the above URL to find the Openshift version you want
	
* **Optional** Enable bash command completion for the oc command, start a new shell with completions enabled

	```bash
	mkdir -p ~/.local/share/bash-completion/completions/
	oc completion bash > ~/.local/share/bash-completion/completions/oc
	exit
	```
	
* Download the pull-secret from https://console.redhat.com/openshift/downloads#tool-pull-secret

	You will need to login with your Red hat credentials, save the file to dev/rhonb/pull-secret.json
	
	**Note:** This file should be kept confidential
	
* Clone the Google microservices demo git repo

	```bash
	cd dev
	git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
	git checkout -t v0.3.1
	```
	Note: The latest version had an issue with local deployments at the time of this writing
	
* Configure AWS CLI

  ```bash
  aws configure
  ```
  Login to your AWS console and create an Access Key https://console.aws.amazon.com/iam/home#/security_credentials and input the credentials into the aws configure prompts. Choose a region that you would like to use. It should look something like this
  
  ```bash
	$ aws configure
	AWS Access Key ID [None]: AKIABOGUSKEY
	AWS Secret Access Key [None]: K73kgBogusKey
	Default region name [None]: ca-central-1
	Default output format [None]: json 
  ```
	The credentials will be saved to ~/.aws/credentials
	
	**Note:** This information should be kept confidential
  
* If you do not already have a ssh key create one and save it into ~/.ssh/

	ed25519 type key recommended, load the key into your ssh-agent.

  ```bash
  ssh-keygen -t ed25519
  ```
	**Note:** The private key should be kept confidential

### Create an AWS Route53 DNS zone

The Openshift installation requires public DNS records. Our organization has a public domain and in order to keep the domain tidy, we'll create a separate subordinated zone for our Openshift installation. This can be created manually in the AWS Route53 console, but I created a small ansible playbook to automate this

* Set environment variables for your public route53 DNS zone and your cluster name

  ```bash
  export BASEZONE=<yourdomain.xyz>
  export CLUSTERNAME=<clustername>
  ```

* Run ansible

  ```bash
  cd rhonb/route53/
  ansible-playbook route53.yaml -D -C -e basezone=$BASEZONE -e prefix=$CLUSTERNAME
  ```
  
  The -C parameter will run the playbook in dry-run mode and will not do any changes. Once you are comfortable with it, run it without -C and a new Route53 zone (clustername.yourdomain.xyz) will be created in AWS.

### Create openshift install-config
* Generate default install-config

  ```bash
  cd .. && mkdir $CLUSTERNAME && cd $CLUSTERNAME
  openshift-install create install-config
  ```
 
* Edit the install-config.yaml in your favorite editor.

	Change the number of compute *replicas:* from 3 to 2
	
	Change *networkType:* from *OpenShiftSDN* to *Calico*
	
### Create and customize manifests

* Generate default manifests

  **Note** Red Hat does not support changes to the generated manifests. For Calico CNI, the company Tigera can offer support.

  ```bash
  openshift-install create manifests
  ```
* Customize manifests with Calico installation

  ```bash
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/01-crd-apiserver.yaml -o manifests/01-crd-apiserver.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/01-crd-installation.yaml -o manifests/01-crd-installation.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/01-crd-imageset.yaml -o manifests/01-crd-imageset.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/01-crd-tigerastatus.yaml -o manifests/01-crd-tigerastatus.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_bgpconfigurations.yaml -o manifests/crd.projectcalico.org_bgpconfigurations.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_bgppeers.yaml -o manifests/crd.projectcalico.org_bgppeers.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_blockaffinities.yaml -o manifests/crd.projectcalico.org_blockaffinities.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_caliconodestatuses.yaml -o manifests/crd.projectcalico.org_caliconodestatuses.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_clusterinformations.yaml -o manifests/crd.projectcalico.org_clusterinformations.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_felixconfigurations.yaml -o manifests/crd.projectcalico.org_felixconfigurations.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_globalnetworkpolicies.yaml -o manifests/crd.projectcalico.org_globalnetworkpolicies.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_globalnetworksets.yaml -o manifests/crd.projectcalico.org_globalnetworksets.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_hostendpoints.yaml -o manifests/crd.projectcalico.org_hostendpoints.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_ipamblocks.yaml -o manifests/crd.projectcalico.org_ipamblocks.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_ipamconfigs.yaml -o manifests/crd.projectcalico.org_ipamconfigs.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_ipamhandles.yaml -o manifests/crd.projectcalico.org_ipamhandles.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_ippools.yaml -o manifests/crd.projectcalico.org_ippools.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_ipreservations.yaml -o manifests/crd.projectcalico.org_ipreservations.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_kubecontrollersconfigurations.yaml -o manifests/crd.projectcalico.org_kubecontrollersconfigurations.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_networkpolicies.yaml -o manifests/crd.projectcalico.org_networkpolicies.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/crds/calico/kdd/crd.projectcalico.org_networksets.yaml -o manifests/crd.projectcalico.org_networksets.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/00-namespace-tigera-operator.yaml -o manifests/00-namespace-tigera-operator.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/02-rolebinding-tigera-operator.yaml -o manifests/02-rolebinding-tigera-operator.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/02-role-tigera-operator.yaml -o manifests/02-role-tigera-operator.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/02-serviceaccount-tigera-operator.yaml -o manifests/02-serviceaccount-tigera-operator.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/02-configmap-calico-resources.yaml -o manifests/02-configmap-calico-resources.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/tigera-operator/02-tigera-operator.yaml -o manifests/02-tigera-operator.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/01-cr-installation.yaml -o manifests/01-cr-installation.yaml
	curl https://docs.projectcalico.org/archive/v3.21/manifests/ocp/01-cr-apiserver.yaml -o manifests/01-cr-apiserver.yaml
  ```
  These are the links to Calico v3.21. Refer to https://projectcalico.docs.tigera.io/getting-started/openshift/installation for the latest version.
  
### Create cluster

This will take some time to run
	
  ```bash
  openshift-install create cluster
  ```
  
### Enable the Calico eBPF dataplane

* Set kubeconfig

	
  ```bash
  export KUBECONFIG=~/dev/rhonb/$CLUSTERNAME/auth/kubeconfig
  ```

* Create ConfigMap

	Create and customize a file with your domain

	```bash
	kind: ConfigMap
	apiVersion: v1
	metadata:
	  name: kubernetes-services-endpoint
	  namespace: tigera-operator
	data:
	  KUBERNETES_SERVICE_HOST: "api-int.<clustername>.<yourdomain.xyz>"
	  KUBERNETES_SERVICE_PORT: "6443"
	```
	
	Create the ConfigMap with oc

	```bash
	oc create -f <filename>
	```
	
	Wait 60 seconds and then restart the tigera-operator
	
	```bash
	oc delete pod -n tigera-operator -l k8s-app=tigera-operator
	```
* Disable kube-proxy

	```bash
	oc patch networks.operator.openshift.io cluster --type merge -p '{"spec":{"deployKubeProxy": false}}'
	```
* Finally switch from the Calico Iptables to eBPF dataplane

	```bash
	kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF", "hostPorts":null}}}'
	```

### Deploy a test application

We'll use the Google microservices demo

* Customize the manifests slightly

	Remove the frontend LoadBalancer service from ~/dev/microservices-demo/kubernetes-manifests.yaml with your favorite editor.
	
	The service starts at line 290 and looks this:
	
	```yaml
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: frontend-external
	spec:
	  type: LoadBalancer
	  selector:
	    app: frontend
	  ports:
	  - name: http
	    port: 80
	    targetPort: 8080
	```
	
* Create project, deploy the demo and expose it
	
	```bash
	oc new-project boutique
	oc create -f ~/dev/microservices-demo/kubernetes-manifests.yaml
	oc expose svc frontend
	```
	
* Test the microservices demo

	Open a browser and go to http://TBD
	
### Clean up

* Delete cluster

	```bash
	cd ~/dev/rhonb/$CLUSTERNAME
	openshift-install destroy cluster
	```
	
* Delete Route53 zone

	```bash
	cd ~/dev/rhonb/route53/
	ansible-playbook route53.yaml -D -C -e state=absent -e basezone=$BASEZONE -e prefix=$CLUSTERNAME
	```