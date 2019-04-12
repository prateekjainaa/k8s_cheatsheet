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
    # delete a rc but keep its pods running
    
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
      
      
  22.       
          
             
  
 

