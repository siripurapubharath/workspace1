   					Kubernetes Test:

Deploying Kubernetes on AWS using EKS.

# 1) Creating a IAM role
	i) enter IAM console 
	ii) click roles
	iii) create role
	iv) select EKS
	v) verify policies
	vi) Review & click Create

# 2) Create VPC
Configure VPC using Cloudformation basic template present in S3 bucket.

	i) Click Create stack
	ii) Select Template
	     a) Select specify amazon s3 url:
  	 https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml

	iii) Need to fill the details as show below
	iv) Review & click Create


# 3) Launching and ec2 instance for master(kubectl):

choose Amazone linux 2 (which has inbuilt tools for our requirement)
choose instance type t2.micro
select the VPC that was created above
subnet-any
public ip enable
select security group
select keypair
review and launch

# 4) Installing and configuring Kubectl & aws-iam-authenticator:

   # i) Kubectl:

curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl

curl -o kubectl.sha256 https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl.sha256

chmod +x ./kubectl

mkdir $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

kubectl version --short --client
Client Version: v1.11.5

   # ii) aws-iam-authenticator

curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/aws-iam-authenticator

curl -o aws-iam-authenticator.sha256 https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/aws-iam-authenticator.sha256

cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc


# 5) Installing & Configuring awscli:

 pip install awscli –upgrade

# aws configure
	AWS Access Key ID [None]: xxxxxxxxxxxxxxxxxxxx
	AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
	Default region name [None]: ap-south-1
	Default output format [None]:

# 6) Creating EKS cluster using awscli:

Command: 
aws eks --region ap-south-1 create-cluster --name msrtest --role-arn arn:aws:iam::xxxx:role/msreksrole --resources-vpc-config subnetIds=subnet-0568b97d067a8d869,subnet-03197c55e02595a52,securityGroupIds=sg-072546f83efba8e8b

Output:
		{
		
     		"cluster": {
        		"status": "CREATING",
        		"name": "msrtest",
        		"certificateAuthority": {},
        		"roleArn": "arn:aws:iam::xxxxxxxxxxxx:role/msreksrole",
        		"resourcesVpcConfig": {
            		"subnetIds": [
                		"subnet-0568b97d067a8d869",
                		"subnet-03197c55e02595a52"
            		],
            		"vpcId": "vpc-046bbbf5bac574eaa",
		            "securityGroupIds": [
                		"sg-072546f83efba8e8b"
            		]
        		},
        		"version": "1.11",
        		"arn": "arn:aws:eks:ap-south-1:xxxxxxxxxxxxxx:cluster/msrtest",
        		"platformVersion": "eks.2",
        		"createdAt": 1552377641.4
    		}
		}


# 7) Creating & configuring kubeconfig file using awscli: 

Command:
aws eks --region ap-south-1 update-kubeconfig --name msrtest

The above command will create .kube folder in $home if dosen’t exist, and a config file in it with below mentioned content.

	apiVersion: v1
	clusters:
	- cluster:
    	certificate-authority-data: LS0-------------------------------------------------------------------------Qo=
    	server: https://---------------------------------.ap-south-1.eks.amazonaws.com
  	name: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
	contexts:
	- context:
    cluster: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
    user: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
  	name: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
	current-context: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
	kind: Config
	preferences: {}
	users:
	- name: arn:aws:eks:ap-south-1:-----------------cluster/msrtest
  	user:
    	exec:
      	apiVersion: client.authentication.k8s.io/v1alpha1
      	args:
      	- token
      	- -i
      	- msrtest
      	command: aws-iam-authenticator

	Command:
	kubectl get svc

	Output:
	NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   13m


# 8) Configure Worker-nodes:
    Configure EKS worker nodes using cloudformation basic template present in S3 bucket:

	i) Click Create stack
	ii) Select Template
                I) Select specify amazon s3 url:
  	 https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-nodegroup.yaml
3) Need to fill the details as show below

Acknowledge:
 I acknowledge that AWS CloudFormation might create IAM resources. 

And click Create

# 9) Attaching worker nodes to the cluster:

To achive above requirement we need to create a yaml file for configmaps.

cat configmaps.yaml
	
	apiVersion: v1
	kind: ConfigMap
	metadata:
  	name: aws-auth
  	namespace: kube-system
	data:
  	mapRoles: |
    	- rolearn: arn:aws:iam::--------------:role/msr-worker-nodes-NodeInstanceRole-xxxxxxxxxx
	      username: system:node:{{EC2PrivateDNSName}}
      	groups:
	        - system:bootstrappers
        	- system:nodes

	Command: # kubectl apply -f configmaps.yaml 
	Output: configmap/aws-auth created

	Command: # kubectl get nodes
	Output:
	NAME                                             STATUS    ROLES     AGE       VERSION
	ip-xxxxxxxxxxxxxx.ap-south-1.compute.internal   Ready     <none>    41m       v1.11.5
	ip-xxxxxxxxxxxxxx.ap-south-1.compute.internal   Ready     <none>    41m       v1.11.5

      
# 10) Tomcat Web Application Server with a sample HTML Page and Persistent volume:

	i) Create a deployment yaml file for tomcat with below content:

	apiVersion: apps/v1beta2
	kind: Deployment
	metadata:
  	name: tomcat-pod
	spec:
  	selector:
    	matchLabels:
      	run: tomcat-pod
  	replicas: 2
  	template:
    	metadata:
      	labels:
	        run: tomcat-pod
    	spec:
      	containers:
      	- name: tomcat
	        image: tomcat:latest
	        volumeMounts:
	        - name: testvolume
	          mountPath: /usr/local/tomcat/webapps/ROOT
        	ports:
	        - containerPort: 8080
	 volumes:
      	  - name: testvolume
 	       persistentVolumeClaim:
        	  claimName: test-pvclaim


create tomcat container with below command:

	kubectl create -f tomcat.yaml
	deployment.apps/tomcat-pod created

ii) create a service yaml file for tomcat with below content:

	apiVersion: v1
	kind: Service
	metadata:
  	name: tomcat-pod
  	labels:
    	run: tomcat-pod
	spec:
  	type: NodePort
  	ports:
  	- port: 8080
    	targetPort: 8080
    	nodePort: 30000
  	type: LoadBalancer
  	selector:
    	run: tomcat-pod

Create service with below command:

	kubectl create -f tomcat-svc.yaml
	service/tomcat-pod created

	kubectl get pods
	NAME                         READY     STATUS    RESTARTS   AGE
	tomcat-pod-5cc66b778-q4qfq   1/1       Running   0          2m
	tomcat-pod-5cc66b778-qqgct   1/1       Running   0          2m

	kubectl get svc
	NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)          AGE
	kubernetes   ClusterIP      10.100.0.1      <none>                                                                     443/TCP          5h
	tomcat-pod   LoadBalancer   10.100.234.49   a4e3367e244d011e981ff0a3bd453b16-1080647442.ap-south-1.elb.amazonaws.com   8080:30000/TCP   53s

	kubectl describe svc tomcat-pod
	Name:                     tomcat-pod
	Namespace:                default
	Labels:                   run=tomcat-pod
	Annotations:              <none>
	Selector:                 run=tomcat-pod
	Type:                     LoadBalancer
	IP:                       10.100.234.49
	LoadBalancer Ingress:     a4e3367e244d011e981ff0a3bd453b16-1080647442.ap-south-1.elb.amazonaws.com
	Port:                     <unset>  8080/TCP
	TargetPort:               8080/TCP
	NodePort:                 <unset>  30000/TCP
	Endpoints:                192.168.159.255:8080,192.168.93.118:8080
	Session Affinity:         None
	External Traffic Policy:  Cluster
	Events:
  	Type    Reason                Age   From                Message
  	----    ------                ----  ----                -------
  	Normal  EnsuringLoadBalancer  1m    service-controller  Ensuring load balancer
  	Normal  EnsuredLoadBalancer   1m    service-controller  Ensured load balancer

Sample HTML file for HTML page:

	<html>
	<header><title>This is title</title></header>
	<body>
	MSRCOSMOS Test
	</body>
	</html>

# 11) Creating Couchdb
	
i) Create a deployment yaml file for couchdb with below content:

	apiVersion: apps/v1beta2
	kind: Deployment
	metadata:
  	name: couchdb-pod
	spec:
  	selector:
    	matchLabels:
      	run: couchdb-pod
  	replicas: 2
  	template:
	    metadata:
	      labels:
        	run: couchdb-pod
    	spec:
      	containers:
	      - name: couchdb
	        image: couchdb:latest
	        ports:
        	- containerPort: 5984

Create couchdb container with below command:

	kubectl create -f couchdb.yaml
	deployment.apps/couchdb-pod created

ii) create a service yaml file for couchdb with below content:

	apiVersion: v1
	kind: Service
	metadata:
  	name: couchdb-pod
  	labels:
    	run: couchdb-pod
	spec:
  	type: NodePort
  	ports:
  	- port: 8080
    	targetPort: 5984
    	nodePort: 30344
  	type: LoadBalancer
  	selector:
    	run: couchdb-pod

Create service for couchdb with below command:

	kubectl create -f couchdb-svc.yaml
	service/couchdb-pod created

	kubectl get pods | grep -i couchdb
	couchdb-pod-56656d69b-4f4ms   1/1       Running   0          19m
	couchdb-pod-56656d69b-sjmj5   1/1       Running   0          19m

	kubectl get svc | grep -i couchdb
	couchdb-pod   LoadBalancer   10.100.25.24     adac3d0a144df11e981ff0a3bd453b16-316676107.ap-south-1.elb.amazonaws.com   8080:30344/TCP   10m

	kubectl describe svc couchdb-pod
	Name:                     couchdb-pod
	Namespace:                default
	Labels:                   run=couchdb-pod
	Annotations:              <none>
	Selector:                 run=couchdb-pod
	Type:                     LoadBalancer
	IP:                       10.100.25.24
	LoadBalancer Ingress:     adac3d0a144df11e981ff0a3bd453b16-316676107.ap-south-1.elb.amazonaws.com
	Port:                     <unset>  8080/TCP
	TargetPort:               5984/TCP
	NodePort:                 <unset>  30344/TCP
	Endpoints:                192.168.102.156:5984,192.168.157.49:5984
	Session Affinity:         None
	External Traffic Policy:  Cluster
	Events:
  	Type    Reason                Age   From                Message
  	----    ------                ----  ----                -------
  	Normal  EnsuringLoadBalancer  9m    service-controller  Ensuring load balancer
  	Normal  EnsuredLoadBalancer   9m    service-controller  Ensured load balancer


# 12) creating Dashboard:

Create a dashboard.yaml file with below content:

	apiVersion: v1
	kind: ServiceAccount
	metadata:
  	labels:
    	k8s-app: kubernetes-dashboard
  	name: kubernetes-dashboard
  	namespace: kube-system
	---
	apiVersion: rbac.authorization.k8s.io/v1beta1
	kind: ClusterRoleBinding
	metadata:
  	name: kubernetes-dashboard
  	labels:
    	k8s-app: kubernetes-dashboard
	roleRef:
  	apiGroup: rbac.authorization.k8s.io
  	kind: ClusterRole
  	name: cluster-admin
	subjects:
	- kind: ServiceAccount
  	name: kubernetes-dashboard
  	namespace: kube-system
	---
	kind: Deployment
	apiVersion: extensions/v1beta1
	metadata:
  	labels:
    	k8s-app: kubernetes-dashboard
  	name: kubernetes-dashboard
  	namespace: kube-system
	spec:
  	replicas: 1
  	revisionHistoryLimit: 10
  	selector:
    	matchLabels:
      	k8s-app: kubernetes-dashboard
  	template:
    	metadata:
      	labels:
        k8s-app: kubernetes-dashboard
    	spec:
      	containers:
      	- name: kubernetes-dashboard
        	image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3
        	ports:
        	- containerPort: 9090
          	protocol: TCP
        	args:
        	livenessProbe:
          	httpGet:
            	path: /
            	port: 9090
          	initialDelaySeconds: 30
          	timeoutSeconds: 30
      	serviceAccountName: kubernetes-dashboard
      	tolerations:
      	- key: node-role.kubernetes.io/master
        	effect: NoSchedule
	---
	kind: Service
	apiVersion: v1
	metadata:
  	labels:
    	k8s-app: kubernetes-dashboard
  	name: kubernetes-dashboard
  	namespace: kube-system
	spec:
  	type: NodePort
  	ports:
  	- port: 80
    	protocol: TCP
    	targetPort: 9090
    	nodePort: 31000
  	type: LoadBalancer
  	selector:
    	k8s-app: kubernetes-dashboard

Create dashboard with below command:

	kubectl create -f dashboard.yaml
	serviceaccount/kubernetes-dashboard created
	clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
	deployment.extensions/kubernetes-dashboard created
	service/kubernetes-dashboard created

	kubectl get pods -n kube-system | grep -i kubernetes
	kubernetes-dashboard-57584d8594-q82nl   1/1       Running   0          2m

	kubectl get svc -n kube-system | grep -i kubernetes-dash
	kubernetes-dashboard   LoadBalancer   10.100.199.128   a9ff74ce9455f11e9aa350a85b7263ff-136063041.ap-south-1.elb.amazonaws.com   80:31100/TCP    2m

	kubectl describe svc -n kube-system kubernetes-dashboard
	Name:                     kubernetes-dashboard
	Namespace:                kube-system
	Labels:                   k8s-app=kubernetes-dashboard
	Annotations:              <none>
	Selector:                 k8s-app=kubernetes-dashboard
	Type:                     LoadBalancer
	IP:                       10.100.199.128
	LoadBalancer Ingress:     a9ff74ce9455f11e9aa350a85b7263ff-136063041.ap-south-1.elb.amazonaws.com
	Port:                     <unset>  80/TCP
	TargetPort:               9090/TCP
	NodePort:                 <unset>  31100/TCP
	Endpoints:                192.168.183.163:9090
	Session Affinity:         None
	External Traffic Policy:  Cluster
	Events:
  	Type    Reason                Age   From                Message
  	----    ------                ----  ----                -------
  	Normal  EnsuringLoadBalancer  2m    service-controller  Ensuring load balancer
  	Normal  EnsuredLoadBalancer   2m    service-controller  Ensured load balancer
	
