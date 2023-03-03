## Cloud Platform : AWS

## 1.) We want to deploy two containers that scale independently from one another.
Container 1: This container runs code that runs a small API that returns users from a database.
Container 2: This container runs code that runs a small API that returns shifts from a database.

solution: we can create two separate Kubernetes Deployments, each with its own replica count and resource limits.

for Container 1:

### step 1: i have created the users-deployment.yaml and users-service.yaml
#### Deployment
```

                    apiVersion: apps/v1
                    kind: Deployment
                    metadata:
                      name: users-deployment
                    spec:
                      replicas: 1
                      strategy:
                        type: RollingUpdate
                        rollingUpdate:
                          maxSurge: 25%
                          maxUnavailable: 25%
                      minReadySeconds: 5
                      revisionHistoryLimit: 10
                      selector:
                        matchLabels:
                          app: users
                      template:
                        metadata:
                          labels:
                            app: users
                        spec:
                          containers:
                          - name: users-container
                            image: kavyapallamreddy/petclinic:5
                            imagePullPolicy: Always
                            resources:
                              limits:
                                cpu: '1'
                                memory: '500Mi'
                              requests:
                                cpu: '1'
                                memory: '500Mi'
                            ports:
                            - containerPort: 8080
                          imagePullSecrets:
                          - name: regcred
```	  
#### Service - LoadBalancer
```
                        apiVersion: v1
                        kind: Service
                        metadata:
                          name: users-service
                          labels:
                            app: users
                        spec:
                          selector:
                            app: users
                          type:  NodePort
                          ports:
                          - nodePort: 32553
                            port: 8080
                            targetPort: 8080
```		
                          
                          
### step 2: 
Using  **kubectl create -f users-deployment.yaml** command we can deploy and using **kubectl create -f users-service.yaml** we can create the service 
            
### step 3: 
If we want to enable the application access for external world , we can use the workernodeip:32553 
            
Similarly for Container 2: shifts deployment
           
### step 4:
		    
#### Deployment
```
                    apiVersion: apps/v1
                    kind: Deployment
                    metadata:
                      name: shifts-deployment
                    spec:
                      replicas: 1
                      strategy:
                        type: RollingUpdate
                        rollingUpdate:
                          maxSurge: 25%
                          maxUnavailable: 25%
                      minReadySeconds: 5
                      revisionHistoryLimit: 10
                      selector:
                        matchLabels:
                          app: shifts
                      template:
                        metadata:
                          labels:
                            app: shifts
                        spec:
                          containers:
                          - name: shifts-container
                            image: stacksimplify/kubenginx:2.0.0
                            imagePullPolicy: Always
                            resources:
                              limits:
                                cpu: '1'
                                memory: '500Mi'
                              requests:
                                cpu: '1'
                                memory: '500Mi'
                            ports:
                            - containerPort: 8085
                          imagePullSecrets:
                          - name: regcred
```		  

#### Service
```
                    apiVersion: v1
                    kind: Service
                    metadata:
                      name: shifts-service
                    spec:
                      type:LoadBalancer # ClusterIp, # NodePort, #LoadBalancer
                      selector:
                        app: shifts
                      ports:
                        - name: http
                          port: 80 # Service Port
                          targetPort: 80 # Container Port
```	  
                            
### step 5: 
Using  **kubectl create -f shifts-deployment.yaml** command we can deploy and using  **kubectl create -f users-service.yaml** we can create the service 

### step 6: 
If we want to enable the application access for external world , we can use dns url
            
            
            
## 2.) For the best user experience auto scale this service when the average CPU reaches 70%.

solution: we can create a horizontal pod autoscaler to specify the minimum and maximum number of pods you want to run, as well as the target CPU utilization or memory utilization of your pods.
in this case : When CPU utilization reaches 70% then we have to increase the pods
         

for example for container 1:
```
            apiVersion: autoscaling/v1
            kind: HorizontalPodAutoscaler
            metadata:
             name: users-hpa
            spec:
             maxReplicas: 5
             minReplicas: 1
             scaleTargetRef:
               apiVersion: apps/v1
               kind: Deployment
               name: users-deployment
             targetCPUUtilizationPercentage: 70
```   
   
   
## 3.)Ensure the deployment can handle rolling deployments and rollbacks.

solution:  By using strategys we can handle rolling deployments and rollbacks and zero down time
```
                    spec: 
                      strategy:
                        type: RollingUpdate
                        rollingUpdate:
                          maxSurge: 25%
                          maxUnavailable: 25%
                      minReadySeconds: 5
                      revisionHistoryLimit: 10
```
					  
			
## 4.) Your development team should not be able to run certain commands on your k8s cluster, but you want them to be able to deploy and roll back. What types of IAM controls do you put in place?
solution: The first thing to do is to create an AWS role named role-developers dedicated to the developers. This role should have at least the following permissions :

```			   
		   "Statement": [
			   {
				   "Action": [
					   "eks:AccessKubernetesApi",
					   "eks:Describe*",
					   "eks:List*",
				   ],
				   "Effect": "Allow",
				   "Resource": "*"
			   },
		   ],
		   "Version": "2012-10-17"
		  }
```	   
Set up RBAC in EKS Cluster
```
		apiVersion: rbac.authorization.k8s.io/v1
		kind: Role
		metadata:
		  name: developers
		rules:
		  - apiGroups:
			- ""
			resources:
			- deployments
			verbs:
			- get
			- list
			- watch
			- create
			- update
```
Now you need to bind these roles to a group with the Kubernetes resource RoleBinding:
		
```
               apiVersion: rbac.authorization.k8s.io/v1
		kind: RoleBinding
		metadata:
		  name: developers
		subjects:
		  - kind: Group
			name: developers   
			namespace: app
			apiGroup: rbac.authorization.k8s.io
		roleRef:
		  kind: Role
		  name: developers
		  apiGroup: rbac.authorization.k8s.io
```  
		  
In an EKS cluster, in the namespace kube-system, you can find a configmap named aws-auth. This configmap is used by EKS to associate an AWS role to a Kubernetes group and thus to give permissions to an AWS role to specific resources in the EKS cluster. 
		
So you have to modify this configmap and add this section:
	
```
		- "groups":
		   - "developers"
		   "rolearn": "arn:aws:iam::1234567890:role/role-developer"
		   "username": "role-developer"	
```		   

## Bonus

### 1.) How would you apply the configs to multiple environments (staging vs production)?
solution: 
1. In most production environments we will have seperate cluster for Testing, Staging and Production. So we can apply the configs to each of these clusters seperately.

2. Another option that is less preferred in my opinion is to maintain a single cluster but separate the workloads within the namespace so you can have a staging namespace a Production namespace etc.,
	  
3. Third option is basically we create one cluster but within the cluster we run kubernetes in multiple virtual environments, each environment will get a separate cluster but it will be present on the same physical cluster		   
					  
### 2.) How would you auto-scale the deployment based on network latency instead of CPU?
solution: we can Autoscale Kubernetes deployments based on custom Prometheus metrics collected from the workloads using CloudWatch Container Insights. It has a custom Kubernetes controller to manage Amazon CloudWatch metric alarms that watch custom metrics data and trigger scaling actions
	
					  



            
          
              
