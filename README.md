geoserver-in-minikube
Configuration files needed to run Geoserver in Minikube

## Description
This project runs the GIS server Geoserver within the Minikube Kubernetes environment.  
The main ingredient is provided by the Dockerized Geoserver created by the fine folks at ThinkWhere:
https://github.com/thinkWhere/GeoServer-Docker.
The above project has been forked to add a few plugins, and adjust how the image is started.

## Setup After Cloning
Create a directory that will host the geoserver data, if you don't have one yet.  If geoserver finds it empty, it will populate this directory with startup data.  I created a directory to hold this in my home directory (my username is tarter):

    mkdir ~/geoserver_data

Next, edit the following line in geoserver-pv.yaml to correspond to where your geoserver data is:

    path: "/hosthome/tarter/geoserver_data/"

The above line would mount the following host directory:

    /home/tarter/geoserver_data/
    
## Minikube Setup
Now, we're ready to set up minikube.  Set the defaults to reserve a bit more cpu, disk, and memory than the defaults.  Also change default driver to virtualbox.  My box has 8 CPUs and 64 GB, so my values may be a bit higher than average, but you need to give Geoserver room to run:

    minikube config set driver virtualbox
    minikube config set cpus 4
    minikube config set memory 24576
    minikube config set disk-size 200GB
    minikube start
    
Enable the ingress addon to provide the interface with the outside world.

    minikube addons enable ingress

Enable the helm tool endpoint within our cluster.  We'll be using this later to install postgresql.

    minikube addons enable helm-tiller

Enable the metrics scaper, which allows showing resource usage in the dashboard.

    minikube addons enable metrics-server

Start the dashboard in the browser so that we can monitor the state of applications as they install, and after.

    minikube dashboard

Add ingress rules for the dashboard so we can access it without issuing a minikube command:

    kubectl apply -f dashboard-ingress.yaml

You'll need to add the address that ingress exposes to your /etc/hosts file.  Execute this command to show that address:

    minikube ip
    
The command above will display status like the following:

    192.168.59.105
    
Next, add an entry in your /etc/hosts file to point to the dashboard service and the soon to be set up geoserver service:

    192.168.59.105  dashboard.info geoserver.info

Once this is complete, the kibernetes dashboard will be accessible at http://dashboard.info/ and the Geoserver instance will be accessible at http://geoserver.info/geoserver .
 
## Geoserver Setup
Now, let's start creating the Geoserver installation.  Create the persistent volume components:

    kubectl apply -f geoserver-pv.yaml
    kubectl apply -f geoserver-pvc.yaml
    
Start the Geoserver deployment, service, and ingress:

    kubectl apply -f geoserver-deployment.yaml
    kubectl apply -f geoserver-service.yaml
    kubectl apply -f geoserver-ingress.yaml
    
Once this finishes spinning up, the Geoserver instance will be accessible at http://geoserver.info/geoserver .

Access the geoserver dashboard at http://geoserver.info/geoserver.  Once on the dashboard, update the base URL or all of the layer previews will be filled with broken links.  Find this from the geoserver menu on the left choose Settings->Global, then change Proxy Base URL on that page to be: "http://geoserver.info/geoserver".

You should probably also change the admin password.  You may access this by going to Security -> Users, Groups, Roles.  Once on that page, click on the Users/Groups tab, then click on the admin user.  On the page that is then brought up, you'll be offered the chance the change the password. 

## Postgresql Setup
Next, add Postgresql to act as the spatial database.  Add the Bitnami helm repository, so we can use their distro.

    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm install my-postgresql bitnami/postgresql

Save the output from the install process.  It includes information about how to access the new instance, and how to set up temporary access outside the cluster.

Let the postgresql installation stabilize.  Once the dashboard shows that the postgres stateful set is showing green status, log on to the pod and enable the PostGIS extensions.  The first command extracts the password; the second command starts the Postgres CLI client:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

    kubectl run my-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:11.14.0-debian-10-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host my-postgresql -U postgres -d postgres -p 5432

Once into the Postgres shell, execute the following commands to enable PostGIS:

    $ postgres=# CREATE EXTENSION postgis;
    CREATE EXTENSION

    $ postgres=# SELECT PostGIS_Version();
            postgis_version            
    ---------------------------------------
     2.5 USE_GEOS=1 USE_PROJ=1 USE_STATS=1
    (1 row)

## To Do

Show how to integrate Postgresql with Geoserver
