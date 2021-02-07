# k8s-vagrant

If you have a bare-metal server or a powerful VM you may easily provision a Kubernetes cluster with several nodes with the simple 
`vagrant up` command using this Vagrantfile.  


It is not a minikube, microk8s or any other specific package. This cluster is installed using standard kubeadm. 
You can use Flannel, Calico or any other CNI. And you can modify it of course, for example to install Rancher. Very simple.  

### What you get

    $ kubectl get no -o wide
    NAME      STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    master    Ready    master   4m8s    v1.19.6   192.168.3.70   <none>        Ubuntu 20.04.1 LTS   5.4.0-64-generic   docker://19.3.8
    worker1   Ready    <none>   2m26s   v1.19.6   192.168.3.71   <none>        Ubuntu 20.04.1 LTS   5.4.0-64-generic   docker://19.3.8
    worker2   Ready    <none>   56s     v1.19.6   192.168.3.72   <none>        Ubuntu 20.04.1 LTS   5.4.0-64-generic   docker://19.3.8

## Configuration

See below the most interesting parameters.  
You may also modify everything else like `--pod-network-cidr=10.253.192.0/18` or even more following vagrant manuals.  

### IP addresses

    IP_NET="192.168.3"
    IP_START=70

Define a subnet for your Kubernetes virtual machines. 
You may have several clusters within the same subnet by starting them from different directories. Just ensure to have proper IP_START in different Vagrantfile.

### Memory and CPU

    DEFAULT_MEM=2048
    WORKER_MEM=3072
    DEFAULT_CPU=2
    WORKER_CPU=2

Default MEM and CPU are applicable to all VMs defined in this file (normally Master, but you can add more).  
Worker MEM and CPU are relevant to workers only

### Workers

    %w{worker1 worker2}.each_with_index do |name, i|

You may specify 1 or more "worker" or rename them.

### Kubernetes version

    apt-get install -qy kubelet=1.19.6-00 kubectl=1.19.6-00 kubeadm=1.19.6-00

I specify 1.19.6-00 because Rancher doesn't support the latest one yet.

## Deploy

Basic commands:

`vagrant up` to deploy for the first time or start from the previous state.  
`vagrant halt` to stop VMs.  
`vagrant destroy` to destroy VMs.  
`vagrant ssh master` ssh into the VM.  

Your directory is mapped to /vagrant/ from inside of VMs.  
`vagrant@master: $ ls -al /vagrant/YAML`

You may change your memory settings after deployment and VMs will start with these new settings. 
It's important, for example, if there is not enough memory to deploy extra agents (admission control, flowlogs, imagescan, runtime, etc).  

## Some extras

### Ingress 

Simple ingress controller installation if needed:

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx

### Access from outside

If you'd like to http to you host (bare-metal) and reach you new Kubernetes ingress you may install an external load balancer or use a `socat` for the simplicity.  

Master VM:

    vagrant@master:~$ kubectl get svc
    NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    ingress-nginx-controller             LoadBalancer   10.102.152.91   <pending>     80:30513/TCP,443:31694/TCP   2m26s
    ingress-nginx-controller-admission   ClusterIP      10.107.47.24    <none>        443/TCP                      2m26s
    kubernetes                           ClusterIP      10.96.0.1       <none>        443/TCP                      68m

Your Host:

    sudo socat tcp-l:80,fork,reuseaddr tcp:192.168.3.71:30513 &
    sudo socat tcp-l:443,fork,reuseaddr tcp:192.168.3.71:31694 &

So connections to your host IP will be successfully forwarded to the worker1 (192.168.3.71) not routable outside of yur host.  

socat is also usefull to map your Host IP:PORT directly to the NodePort of your worker.  

### DVWA
 
`YAML\dvwa.yaml`  
Damn Vulnerable Web App (DVWA) is a PHP/MySQL web application that is damn vulnerable. Its main goals are to be an aid for security professionals to test their skills and tools in a legal environment, help web developers better understand the processes of securing web applications and aid teachers/students to teach/learn web application security in a class room environment.  
https://dvwa.co.uk/  

Danger!!! Do not expose publicly!!!

### Sock Shop

`YAML\weave-sock-shop.yaml`
A Microservices Demo Application.  
Sock Shop simulates the user-facing part of an e-commerce website that sells socks. It is intended to aid the demonstration and testing of microservice and cloud native technologies.  
https://microservices-demo.github.io/  


### For references only

`join.sh` was generated during the master installation and used by workers to join the cluster.

`*.log` are for references only as well
