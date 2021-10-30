# geonode-geoserver-in-minikube
Configuration files needed to run Geoserver in Minikube

## Description
This project runs the GIS server Geoserver within the Minikube Kubernetes environment.  
The main ingredient is provided by the Dockerized Geoserver created by the fine folks at Geonode:
https://hub.docker.com/r/geonode/geoserver .


## Setup After Cloning
Dowload the latest data-2.xx.x.zip file from https://build.geo-solutions.it/geonode/geoserver/latest/ .
Create a directory named "geoserver" where this repository was cloned and unzip the data there:

    mkdir geoserver
    unzip ~/Downloads/data-2.15.1-osgeolive.zip -d ./geoserver/

Next, edit the following line in geoserver-pv.yaml to correspond to where your geoserver data is:

    path: "/hosthome/tarter/GIS/geonode/geonode-geoserver-in-minikube/geoserver/data_osgeolive/"

The above line would mount the following host firectory:

    /home/tarter/GIS/geonode/geonode-geoserver-in-minikube/geoserver/data_osgeolive/
    
Now, we're ready to set up minikube.  Set the defaults to reserve a bit more cpu and memory, and change default driver to virtualbox:

    minikube config set driver virtualbox
    minikube start --cpus 4 --memory 8192
    
Enable the ingress addon to provide the interface with the outside world.

    minikube addons enable ingress

Start the dashboard so that the dashboard elements will be installed.

    minikube dashboard
    
Next, create the persistent volume components:

    kubectl apply -f geoserver-pv.yaml
    kubectl apply -f geoserver-pvc.yaml
    
Start the geoserver deployment, service, and ingress:

    kubectl apply -f geoserver-deployment.yaml
    kubectl apply -f geoserver-service.yaml
    kubectl apply -f geoserver-ingress.yaml

Add ingress rules for the dashboard

    kubectl apply -f dashboard-ingress.yaml
    
You'll need to add the address that ingress exposes to your /etc/hosts file.  Execute this command to show the address:

    kubectl get ingress -n kubernetes-dashboard
    
The command above will display status like the following:

    NAME                CLASS    HOSTS            ADDRESS          PORTS   AGE
    dashboard-ingress   <none>   dashboard.info   192.168.99.110   80      108s
    
Next, add an entry in your /etc/hosts file to point to the dashboard service and the geoserver service:

    192.168.99.110  geoserver.info dashboard.info

Once this is complete, the kibernetes dashboard will be accessible at http://dashboard.info/ and the Geoserver instance will be accessible at http://geoserver.info/geoserver .
