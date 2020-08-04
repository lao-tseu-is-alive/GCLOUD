# GCLOUD
My Google Cloud experiments while doing this quest :
[Getting Started: Create and Manage Cloud Resources](https://google.qwiklabs.com/quests/120)

_on your Linux terminal you can install the SDK:_ 
```bash
sudo apt-get install google-cloud-sdk
```

_you now can use gcloud interactive with:_ 
```bash
gcloud beta interactive
```

##### Main URLS for Compute Ressources

+ [FREE TIER](https://cloud.google.com/free)
+ [Google Cloud Pricing Calculator](https://cloud.google.com/products/calculator#id=)
+ [Machine Types](https://cloud.google.com/compute/docs/machine-types)

##### Setting your default zone with gcloud
+ [Regions and Zone](https://cloud.google.com/compute/docs/regions-zones)
+ [Global, regional and zonal resources](https://cloud.google.com/compute/docs/regions-zones/global-regional-zonal-resources)

Resources that live in a zone, such as virtual machine instances or zonal persistent disks, are referred to as zonal resources. Other resources, like static external IP addresses, are regional. Regional resources can be used by any resource in that region, regardless of zone, while zonal resources can only be used by other resources in the same zone.

For example, to attach a zonal persistent disk to an instance, both resources must be in the same zone. Similarly, if you want to assign a static IP address to an instance, the instance must be in the same region as the static IP address.

_This is how to set a default zone :_
```bash
export ZONE=us-east1-c
gcloud config set compute/zone $ZONE  
```

##### Creating VM with gcloud
to create a latest Debian image with a  "2-CPU, 7.5GB RAM" instance *n1-standard-2*
on the selected zone : 
```bash
export VMNAME=myserver
gcloud compute instances create $VMNAME \
--machine-type n1-standard-2 --zone $ZONE  
```

_to connect to this VM with ssh :_
```bash
gcloud compute ssh $yourVmName --zone us-central1-c  
```

##### Creating KUBERNETES Cluster with gcloud
- set default zone
- create cluster
- get authentication credentials for the cluster
- Deploying a containerized application to the cluster
- Create a Kubernetes Service, to expose your application to external traffic 
```bash
gcloud config set compute/zone us-east1-c
gcloud container clusters create your-k8s-cluster
gcloud container clusters get-credentials your-k8s-cluster --zone us-east1-c
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
kubectl get service
````
 now you can check that all is working by navigating to the  http://[EXTERNAL-IP]:8080     
      
  ##### Creating multiple web server instances
  To simulate serving from a cluster of machines, 
  create a simple cluster of Nginx web servers to serve static content using
   [Instance Templates](https://cloud.google.com/compute/docs/instance-templates)
   and [Managed Instance Groups](https://cloud.google.com/compute/docs/instance-groups/).
 
 To create a Nginx web server cluster :
 
_let's create a startup script to be used in all vm:_ 
```bash
     cat << EOF > startup.sh
     #! /bin/bash
     apt-get update
     apt-get install -y nginx
     service nginx start
     sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
     EOF
````

_Create an instance template, which uses the startup script:_
```bash
gcloud config set compute/zone $ZONE
gcloud compute instance-templates create nginx-template \
         --metadata-from-file startup-script=startup.sh  
```   

_Create a target pool to allow a single access point to all the instances in a group:_
```bash
gcloud compute target-pools create nginx-pool  
```   

_Create a managed instance group using the instance template:_
```bash
gcloud compute instance-groups managed create nginx-group \
         --base-instance-name nginx \
         --size 2 \
         --template nginx-template \
         --target-pool nginx-pool  
```   
This creates 2 virtual machine instances with names that are prefixed with nginx-. This may take a couple of minutes.

_List the compute engine instances and you should see all of the instances created:_   
```bash
gcloud compute instances list  
```    
_Now configure a firewall so that you can connect to the machines on port 80 via the EXTERNAL_IP addresses:_
```bash
gcloud compute instance-groups managed create nginx-group   
```  

 ##### Create a HTTP(s) Load Balancer
 
_First, create a [health check](https://cloud.google.com/compute/docs/load-balancing/health-checks). 
Health checks verify that the instance is responding to HTTP or HTTPS traffic:_   
```bash
gcloud compute http-health-checks create http-basic-check  
```    

_Define an HTTP service and map a port name to the relevant port for the instance group. 
Now the load balancing service can forward traffic to the named port:_   
```bash
gcloud compute instance-groups managed \
       set-named-ports nginx-group \
       --named-ports http:80  
```    

_Create a [backend service](https://cloud.google.com/compute/docs/reference/latest/backendServices):_
```bash
gcloud compute backend-services create nginx-backend \
      --protocol HTTP --http-health-checks http-basic-check --global
maybe use also : 
--instance-group-zone $ZONE  
```  
_Add the instance group into the backend service:_
```bash
gcloud compute backend-services add-backend nginx-backend \
    --instance-group nginx-group \
    --instance-group-zone $ZONE \
    --global   
```

_Create a default URL map that directs all incoming requests to all your instances:_
```bash
gcloud compute url-maps create web-map \
    --default-service nginx-backend
```

To direct traffic to different instances based on the URL being requested, see [content-based routing](https://cloud.google.com/compute/docs/load-balancing/http/content-based-example).

_Create a target HTTP proxy to route requests to your URL map:_
```bash
gcloud compute target-http-proxies create http-lb-proxy --url-map web-map
```

_Create a target HTTP proxy to route requests to your URL map:_
```bash
gcloud compute target-http-proxies create http-lb-proxy --url-map web-map
```
 
_Create a global forwarding rule to handle and route incoming requests:_
```bash
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80
```

A forwarding rule sends traffic to a specific target HTTP or HTTPS proxy depending on the IP address, IP protocol, and port specified. The global forwarding rule does not support multiple ports. 

_Create a target HTTP proxy to route requests to your URL map:_
```bash
gcloud compute forwarding-rules list
```

Take note of the http-content-rule IP_ADDRESS for the forwarding rule.

From the browser, you should be able to connect to http://IP_ADDRESS/. 
It may take three to five minutes. If you do not connect, wait a minute then reload the browser.

after that by pressing Refresh various times 