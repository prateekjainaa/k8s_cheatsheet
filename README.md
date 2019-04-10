# k8s_cheatsheet
notes on k8s

1. Enable auto-completion on linux. In .bash_rc file

 source <(kubectl completion bash | sed s/kubectl/k/g)

Now, we can also use k inplace of kubectl.

2. To run a pod
  
   k run mynginx --image=nginx --generator=run-pod/v1 -n myNS
   
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
    
17. 
