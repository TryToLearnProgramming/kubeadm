KubeADM setup in VirtualBox

in this test node details are =>

	master-node = [master, XXXX, 192.168.31.210]
	worker-node1 = [node1, YYYY, 192.168.31.211]
	#worker-node2 = [node2, ZZZZ, 192.168.31.212]
	
device requirements(minimum)- 

	master 2vCPU, 2GB memory, 30GB storage
	worker 1vCPU, 2GB memory, 30GB storage

Step 1:
This step is same for master and worker nodes -

	Disable swap memory -
		sudo swapoff -a
		sudo nano /etc/fstab => remove or comment-out swap line
		
	sudo apt-get update
	sudo apt install docker.io # or follow docker official doc
	sudo systemctl enable docker
	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
	sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
	sudo apt-get update
	sudo apt install kubeadm
	
Step 2:
Clone this VM and create other 2 VM for node
	create different users in there like node1 and node2
	
	sudo adduser <user_name>
	sudo usermod -aG sudo <user_name>
	
Step 3:
After successfully perform step1 and 2 -
	Innitilize the master node
		
	sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<masternode_ip>


Step 4:
Join worker node with master node -> after run kubeadm init it will return a command to a token
	
	kubeadm join 192.168.31.210:6443 --token xdjkhz.dc756fla4ln9u2g6 --discovery-token-ca-cert-hash sha256:ccd2a0a0e1dc6deed2a6b6d712751e535ae706dd901466910e91e596fb730999
	
Step 5:
After successfully join them, then need to install a CN. Caloco is a prefarable one

	kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
	
*****Wait for master and worker node to become in ready state ------- if take too long then restert VMs
	
Step 6:
Install MetalLB (Metal LoadBalancer)
for routing traffic to the internet.
	
	kubectl get configmap kube-proxy -n kube-system -o yaml | \
	sed -e "s/strictARP: false/strictARP: true/" | \
	kubectl apply -f - -n kube-system
	
	kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
		
Configure IP pools in metalLB (yaml):
		
	apiVersion: metallb.io/v1beta1
	kind: IPAddressPool
	metadata:
		name: first-pool
		namespace: metallb-system
	spec:
		addresses:
		- 192.168.31.203-192.168.31.209 #IP addresses other than node IPs not in DHCP range
		
	---
	
	apiVersion: metallb.io/v1beta1
	kind: L2Advertisement
	metadata:
		name: l2-first-pool
		namespace: metallb-system
	spec:
		ipAddressPools:
		- first-pool
	
Step 7:
Install nginx Ingress : 
	
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml
	
Step 8:
Deploy the application.
	e.g: ArgoCD with ingress
		
	https://argo-cd.readthedocs.io/en/stable/getting_started/ # install argocd
		
Ingress Manifest (yaml):
			
	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
		name: argocd-server-ingress
		namespace: argocd
		annotations:
		nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
	spec:
		ingressClassName: nginx
		rules:
		- host: argo.mysite.sami ## specify your domain
		http:
			paths:
			- path: /
			pathType: Prefix
			backend:
				service:
				name: argocd-server
				port:
					name: https


to assign static DHCP ip to nodes -
	sudo nano /etc/netplan/*.yaml

	network:
		version: 2
		ethernets:
			enp0s3:
				dhcp4: no
				address:
					- 192.168.x.x/24 # ip you want to assign Node
				gateway4: 192.168.x.x
				nameservers:
					addresses: [192.168.x.x] # same as gateway

	sudo netplan apply
	
	
	
REFERANCES:

	https://argo-cd.readthedocs.io/en/stable/getting_started/
	https://metallb.universe.tf/installation/
	https://www.adaltas.com/en/2022/09/08/kubernetes-metallb-nginx/

