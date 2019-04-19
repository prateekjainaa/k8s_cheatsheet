# k8s_cheatsheet
notes on k8s

1. Enable auto-completion on linux. In .bash_rc file

 source <(kubectl completion bash | sed s/kubectl/k/g)

Now, we can also use k inplace of kubectl.

2. To run a pod
  
   k run mynginx --image=nginx --generator=run-pod/v1 -n myNS
   
   -- better way
   
    k create deployment mydep2 --image=nginx -n myNS
   
3. To list pods in myNS namespace
  
   k get po -o wide -n myNS
   
4. To see more info. about this pod

    k describe po mynginx -n myNS
    
5. To get deatils about yaml parts

    k explain pod spec.metadata
    
6. To expose as service for example,

    k expose rs mynginx --name nginx-http --type=LoadBalancer  -n myNS
    
    k get svc -n myNS

7.  To scale up rs or rc,

    k scale rs mynignx-http --replicas=3
    
8.  To get dashboard info.

    k cluster-info | grep dashboard
    
    on minikube, 
    
    minikube dashboard
    
9.  To get nodes info.

    k get nodes 
    
10. Main parts of pod definition: spec, metadata, status

    kubectl explain pod.spec
    
11. To see logs of container

   k logs mynginx -c <name_of_container>
    
12. Sending requests/calls to pod:

  a. expose as service
  b. port forwarding ( k port-forward mynginx 80:80 )
  c. start proxy with 'kubectl proxy'
  
13. pod labels

  a. k get po --show-labels -n myNS
  b. k get po -L myLabel
  c. k get po -l '!myLabel' # expression in quotes 
  d. k label po mynginx newLabel=newVal
  e. k label po mynginx existingLabel=newVal --overwrite
  f. k get po -l 'run=myapp,lab=test' # expression in quotes 
  
14. node labels

  a. k label node myNode ssd=true
  b. nodeSelector can be used in pod spec to choose nodes for scheduling.
  c. choose a specific node by label 'kubernetes.io/hostname'
  
15. pod annotations:
    k get po mynginx
    k annotate pod mynginx abc.com/foo="bar"
    
    k delete po mynginx # delete a pod
    
16. Namespaces

    k get ns
    k get po -n kube-system
    
    kube-system, kube-public, default
    
    k delete ns myNS # will delete ns and everything under it.
    k delete po --all # will delete all pods in my current ns.
    k delete all --all # will delete everything (except some like secrets) inside current ns.
    
## ---------------------------------------
17. livenessProbe - httpGet, exec, tcpSocket
    Exit code is (128 + x) for example 137 = (128 + 9). 9 is ``sigKill`` signal.
    When a container is killed, a new container is created. same container is not restarted.
    Always use ``initialDelaySeconds`` to account in App's startup time.
    
    keep probes light for example dont spawn a new jvm while checking java based app.
    
    Don't specify selector in rc yaml, k8s will extract it from template and will keep file simple.
    
    to turn off network: ``sudo ifconfig eth0 down``
    if a node becomes unreachable in network then pods on it are marked with ``UNKNOWN`` status.
    
    By updating pod's label it can be removed or added to scope of rc.
    
    k edit rc mynginx -n myNS
     delete a rc but keep its pods running
    
    k delete rc mynginx --cascade=false
    
    rs has both matchLabels and matchExpressions in selector.
    
 18. DaemonSet to run 1 pod per node
 
    Note: To run DaemonSet on selective nodes, use nodeSelector in pod template  
    
19. Job - Where you want to run a task that terminates after completing its work.

    k create job myjob --image=busybox -n myNS -- ls
    
    Running multiple pod instances and running them in sequence or in parallel is controlled by ``completions`` and ``parallelism`` in job spec. A job can be scaled like rs (it increases parallel). Job's time can be limited by activeDeadLineSeconds. 
    
 20. CronJob is similar to job with spec.schedule and startedDeadlineSeconds properties.
 
 21. Service -> sessionAffinity: clientIP | None
 
     Pods can got IP and port of services by looking up environment variables. for example, 
     # MYNGINX_SERVICE_HOST=10.22.33.201
     
     Whether the pod uses internal DNS service is configurable by ``dnsPolicy`` in pod.spec
     
     Service also gets a FQDN - <service_name>.<namespace>.svc.cluster.local
 
     Pod also stores/caches DNS information in /etc/resolv.conf
     
     ## You can't ping service IP
     
     Service has endpoints. You can point a service to external service like DB - 
       a. Create a service w/o pod selector.
       b. i. By manually creating EndPoints (its name should exactly match with service name), or
          ii. Set the ``type`` field of service to ExternalName like,
          
            kind: Service
            spec:
              type: ExternalName
              externalName: someapi.outside.com
              
      c. Service can be exposed to external clients via NodePort, LoadBalancer and Ingress.             
          You can use ``externalTrafficPolicy`` field of service to avoid additional network hops but it might result in hanged requests to nodes which dont have those pods.
          By Setting clusterIP to ``None``, we can create a headless service.
          
      d. Add ``service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"`` annotation to service for listing all pods added to service irrespective of their status.
      
      e. If you can’t even access your app through the pod’s IP, make sure your app isn’t only binding to localhost.
      
      
  22.  Volumes: 
       At pod level
       
         volumes:
           - name: myhtmlDir
             emptyDir/gitRepo/...
           
       At container level
       
         volumeMounts:
           - name: myhtmlDir
             mountPath: /var/www/html
             readOnly: true
      
      Note: use emptyDir.medium as Memory to stop writing anything to disk.
      
  Use ``hostPath`` volume to share/access filesystem of node in pod.
  
  To use persistent storage for DB like mysql, mongodb etc. use gcePersistentDisk/awsElasticBlockStore/azureDisk or nfs.
  
  ``configMap, secret, downwardAPI``—Special types of volumes used to expose certain Kubernetes resources and cluster information to the pod.
  
  But this kind of information should be decoupled from pod definition via PVC and PV.
  
  While creating PV, admin needs to tell its capacity and whether it can be read by or written by single or multiple pods. Also, specify what to do when PV is released.
  
``kind: PersistentVolume
  metadata:
    name: mongodb-pv
  spec:
    capacity:
      storage: 1Gi
      accessModes:
      - ReadWriteOnce
      - ReadOnlyMany
      persistentVolumeReclaimPolicy: Retain
      gcePersistentDisk:
        pdName: mongodb
        fsType: ext4         
  ``
  
  PV are not bound to namespace. They are cluster level resources. But PVC are namespace scoped and can only be used by pods of same namespace.
  
  Now, create PVCs like 

````
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
  ````
  
This can be reffered into Pod as,

````
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
````

PV status could be Available/Bound/Pending.

 the only way to manually recycle the PersistentVolume to make it available again is to delete and recreate the PersistentVolume         resource.

# Reclaiming PersistentVolumes automatically
Two other possible reclaim policies exist: Recycle and Delete. The first one deletes the volume’s contents and makes the volume available to be claimed again. This way, the PersistentVolume can be reused multiple times by different PersistentVolumeClaims and different pods. The Delete policy, on the other hand, deletes the underlying storage. 

# DYNAMIC PROVISIONING OF PERSISTENTVOLUMES

The cluster admin, instead of creating PersistentVolumes, can deploy a PersistentVolume provisioner and define one or more         StorageClass objects to let users choose what type of PersistentVolume they want. The users can refer to the StorageClass in their PersistentVolumeClaims and the provisioner will take that into account when provisioning the persistent storage.

``StorageClass is not namespaced.``

# Specifying an empty string as the storage class name ensures the PVC binds to a pre-provisioned PV instead of dynamically provisioning a new one.

23. ConfigMaps and Secrets

 Regardless if you’re using a ConfigMap to store configuration data or not, you can configure your apps by                                    Passing command-line arguments to containers
     Setting custom environment variables for each container
     Mounting configuration files into containers through a special type of volume
  
In a Dockerfile, two instructions define the two parts:

  ENTRYPOINT defines the executable invoked when the container is started.
  CMD specifies the arguments that get passed to the ENTRYPOINT.

Overriding the command and arguments in Kubernetes:
````
kind: Pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
````
 
 Specifying environment variables in a container definition:
 ````
 containers:
 - image: myimage
   env:                            
   - name: INTERVAL                
     value: "30"                   
   name: html-generator
````

Referring to other environment variables in a variable’s value
````
env:
- name: FIRST_VAR
  value: "foo"
- name: SECOND_VAR
  value: "$(FIRST_VAR)bar"
  ````
  
  To reuse the same pod definition in multiple environments, it makes sense to decouple the configuration from the pod descriptor.
  
  ``kubectl create configmap fortune-config --from-literal=sleep-interval=25 --from-literal=bar=baz``
  
  ``kubectl create configmap my-config --from-file=config-file.conf``
  
  Now this can be used in pod to inject values
  
  ````
  spec:
  containers:
  - image: myniginx:env
    env:                             
    - name: INTERVAL                 
      valueFrom:                     
        configMapKeyRef:             
          name: my-config       
          key: sleep-interval
  ````
  
  You can also mark a reference to a ConfigMap as optional (by setting configMapKeyRef.optional: true). In that case, the container starts even if the ConfigMap doesn’t exist.
  
  You can expose all keys inside configMap as environment variables by using the envFrom attribute, instead of env the way
  ````
  spec:
  containers:
  - image: some-image
    envFrom:                      
    - prefix: CONFIG_             
      configMapRef:               
        name: my-config      
   ````
   
   Passing a ConfigMap entry as a command-line argument
  ```` 
  spec:
  containers:
  - image: mynginx
    env:                               
    - name: INTERVAL                   
      valueFrom:                       
        configMapKeyRef:               
          name: my-config         
          key: sleep-interval          
    args: ["$(INTERVAL)"]            
   ````
    
    # Using a configMap volume to expose ConfigMap entries as files
    
    Passing configuration options as environment variables or command-line arguments is usually used for short variable values. A ConfigMap, as you’ve seen, can also contain whole config files. When you want to expose those to the container, you can use one of the special volume types configMap volume.
    
    ``kubectl create configmap fortune-config --from-file=configmap-files``
    Here, configmap-files directory contains file named 'sleep-interval' whose value is 25 and another file called my-nginx-config.conf.
    
    ````
    spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    ...
    - name: config
      mountPath: /etc/nginx/conf.d      
      readOnly: true
    ...
  volumes:
  ...
  - name: config
    configMap:                          
      name: my-config              
    ````
    
    To define which entries should be exposed as files in a configMap volume, use the volume’s items attribute as shown in the following listing.
    ````
    volumes:
  - name: config
    configMap:
      name: my-config
      items:                             
      - key: my-nginx-config.conf        
        path: gzip.conf                  
   ````     
   
    The directory then only contains the files from the mounted filesystem, whereas the original files in that directory are inaccessible for as long as the filesystem is mounted.
    To add individual files from a ConfigMap into an existing directory without hiding existing files stored in it.
    An additional 'subPath' property on the volumeMount allows you to mount either a single file or a single directory from the volume instead of mounting the whole volume. 
  ````  
  spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/someconfig.conf    
      subPath: myconfig.conf    
    ````
  
  # Always remember to use --record flag when creating deployment
  
   `` kubectl create -f mydeployment.yaml --record ``
   It records the command in revision history.
    
    `` kubectl rollout status deployment mydeployment ``
    
    `` kubectl rollout undo deployment mydeployment ``
    
    `` kubectl rollout undo deployment mydeployment --to-revision=2 ``
    
    `` kubectl rollout pause deployment mydeployment ``
    
    `` kubectl rollout resume deployment mydeployment ``
    
    ``minReadySeconds`` of Deployment is time for which pod should be ready before making it available.
    
    
 24. Stateful sets
 
   stateful vs replicas sets can be understood via pet vs cattle analogy.
   
   stateful set requires you to create a corresponding governing headless service that's used to provide the actual network identity to each pod. Through this service each pod gets its own DNS entry.
   You get pods name appended with numeric index like A-0, A-1, A-2. 
   StatefulSets scale down only one pod at a time. Also, they also never permit scale down operations if any of the instances are unhealthy. These also need to use same storage when rescheduled. Because PVC map to PV one-to-one, each pod from statefulset need to have its own PVC to have its own PV. So, you have to stamp out PVC's in similar way as pod template using volume-claim templates.
   
   Scaling up a StatefulSet by one creates two or more API objects (the pod and one or more PersistentVolumeClaims referenced by the pod). Scaling down, however, deletes only the pod, leaving the claims alone. The fact that the PersistentVolumeClaim remains after a scale-down means a subsequent scale-up can reattach the same claim along with the bound PersistentVolume and its contents to the new pod instance.
   StatefulSet must be absolutely certain that a pod is no longer running before it can create a replacement pod. This has a big effect on how node failures are handled. Kubernetes must thus take great care to ensure two stateful pod instances are never running with the same identity and are bound to the same PersistentVolumeClaim. A StatefulSet must guarantee at-most-one semantics for stateful pod instances.
   
   The StatefulSet manifest isn’t that different from ReplicaSet or Deployment manifests you’ve created so far. What’s new is the volumeClaimTemplates list.
   
   ````
       spec:
      containers:
      - name: mypod
        image: nginx
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data                 
          mountPath: /var/data       
  volumeClaimTemplates:
  - metadata:                        
      name: data                     
    spec:                            
      resources:                     
        requests:                    
          storage: 1Mi               
      accessModes:                   
      - ReadWriteOnce     
  ````    
   
   The second pod will be created only after the first one is up and ready. StatefulSets behave this way because certain clustered stateful apps are sensitive to race conditions if two or more cluster members come up at the same time.
   
 ------
 generate key and cert file for https/ssl
 
 openssl genrsa -out tls.key 2048
 openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com
 --------------
 
 
   
   
   
   
    
