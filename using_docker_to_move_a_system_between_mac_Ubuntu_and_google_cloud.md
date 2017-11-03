This howto describes how to get a docker container moved between Ubuntu Linux, Mac and Google cloud. It demonstrates the beauty of how docker makes it possible to move a complex system between several platforms.  
You can use the howto on any other docker image.  So even if you are not installing discourse you can use it.

I wanted to set up discourse.org on a VM hosted at Google compute engine. It was not trivial and I want to share the journey so that you can save time.

This howto includes 3 hosts
- Mac -- where I do my normal work
- localubuntu -- Ubuntu running in VirtualBox on my Mac
- urbalurbahost -- A VM instance running in Google Compute Engine

#### About discourse.org
[discourse.org](http://discourse.org) isthe best discussion system you can get. It is open source and if you plan to create a community then this is what to get. If you, like me, just want the discussion functionality and not the pain of setting up servers. Then running discourse in a docker container is the best thing.  This is described here: 
discourse_docker https://github.com/discourse/discourse_docker

BUT. there are some tweaks to it. And this is how to solve them. 

Local development on my Mac

I normally use docker on my mac and wanted to build the discourse_docker container on my mac. I found out that that was not possible because of some errors in the launcher script.
This lead to having to install Ubuntu and getting containers moved between different hosts.


## Ubuntu in VirtualBox on my Mac
Solution to this was to set up [VirtualBox](http://VirtualBox.org) and install [Ubuntu](https://www.ubuntu.com/download/server) on it (named it localubuntu).
This worked just fine and I was able to build and run discourse_docker container inside my ubuntu under VirtualBox. (Use Bridged network adapter so that localubuntu has a IP address on your network)

### Google cloud SDK on Ubuntu
In order to get discourse_docker to google cloud I needed to push it to google docker registry. For this I needed to install google cloud SDK on localubuntu. 
Used this to install https://cloud.google.com/sdk/docs/quickstart-linux on localubuntu.
(worked fine, but install script did not update path to gcloud so I had to do it manually)

### Pushing discourse_docker container to google cloud from localubuntu 
Get our project ID
```
PROJECT_ID="$(gcloud config get-value project -q)"
```
List the images on localubuntu
```
docker images
```

Tag the discourse image “urbalurba-discourse:v1”
```
docker image tag local_discourse/app:latest gcr.io/${PROJECT_ID}/urbalurba-discourse:v1
```

Push it to the google docker registry
```
gcloud docker -- push gcr.io/${PROJECT_ID}/urbalurba-discourse:v1
```
The container is HUGE so be patient.

## Getting the container back to my Mac
Next I want to run the container on my mac. That was where I wanted it in the first place.

### Google cloud SDK on Mac
Here I also had to install google cloud SDK
https://cloud.google.com/sdk/docs/quickstart-mac-os-x
(also here the script did not update the path to gcloud so it needs to be done manually)
 
### Pulling discourse_docker container from google cloud to my Mac
Pull the image to the mac
```
gcloud docker -- pull gcr.io/urbalurba-184319/urbalurba-discourse:v1
```
(I had login problems first, so make sure you are logged in with the same user as on localubuntu)

### Copy data files from localubuntu to Mac host

Prepare Mac host data disk
* Created the dir  /Users/tec/dockerdisk/urbalurba-discourse/shared
* Created the dir /Users/tec/dockerdisk/vmdisk

On localubuntu go to the dir that has the datafiles
```
cd /var/discourse/shared
```
 tar the directory
 ```
sudo tar -cvf standalone_dir.tar standalone
```
On the Mac host i have a temp dir. Go there
```
cd dockerdisk/vmdisk
```
Copy the files from localubuntu
```
scp your_username@172.16.1.186:/var/discourse/shared/standalone_dir.tar .
```
(ip is the address of the localubuntu. Make sure u have set up network to bridge in VirtualBox)

In Mac finder click on the standalone_dir.tar file to extract it. 
Copy the extracted dir to the prepared directory
* /Users/tec/dockerdisk/urbalurba-discourse/shared 


### Run the container on the Mac
Now that we have the container on the mac it’s time to run it. I copy the docker run with all its parameters from localubuntu. I need to change the following parameters.

* DOCKER_HOST_IP=docker.for.mac.localhost 
* The image name : gcr.io/urbalurba-184319/urbalurba-discourse:v1

And the volumes 
* -v /Users/tec/dockerdisk/urbalurba-discourse/shared/standalone:/shared 
* -v /Users/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log:/var/log 

I have removed passwords and stuff. Then the docker command looks like this

> docker run -d --restart=always -e LANG=en_US.UTF-8 -e RAILS_ENV=production -e UNICORN_WORKERS=2 -e UNICORN_SIDEKIQS=1 -e RUBY_GLOBAL_METHOD_CACHE_SIZE=131072 -e RUBY_GC_HEAP_GROWTH_MAX_SLOTS=40000 -e RUBY_GC_HEAP_INIT_SLOTS=400000 -e RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=1.5 -e DISCOURSE_DB_SOCKET=/var/run/postgresql -e DISCOURSE_DB_HOST= -e DISCOURSE_DB_PORT= -e DISCOURSE_HOSTNAME=discourse.urbalurba.no -e DISCOURSE_DEVELOPER_EMAILS=notme@urbalurba.no -e DISCOURSE_SMTP_ADDRESS=smtp.elasticemail.com -e DISCOURSE_SMTP_PORT=2525 -e DISCOURSE_SMTP_USER_NAME=notme@urbalurba.no -e DISCOURSE_SMTP_PASSWORD=I-will-not-give-you-this_:) -h urbadics -e DOCKER_HOST_IP=docker.for.mac.localhost --name urbalurbadiscourse -t -p 80:80 -p 443:443 -v /Users/tec/dockerdisk/urbalurba-discourse/shared/standalone:/shared -v /Users/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log:/var/log gcr.io/urbalurba-184319/urbalurba-discourse:v1 /sbin/boot


The container should now start and you can check that it is running by 
```
docker ps
```
Now open http://localhost on your mac in a browser of your choice


This is so cool. I can now change stuff (install plugins and so on) in localubuntu and do a push and a pull on the mac. 



## Deploying discourse_docker container to google cloud
I have a instance (a VM in google compute engine) that is set up using this howto https://cloud.google.com/container-optimized-os/docs/how-to/create-configure-instance
(the “Creating an instance using the Google Cloud Platform Console” in the howto does not work) so just do this:


### Create a instance on Google compute engine

List available VMs
```
gcloud compute images list \
     --project cos-cloud \
     --no-standard-images
```
You get something like this:

NAME |                     PROJECT |   FAMILY |     DEPRECATED | STATUS
cos-beta-62-9901-50-0  |   cos-cloud | cos-beta |            |   READY
cos-dev-63-10032-4-0   |   cos-cloud | cos-dev  |            |   READY
cos-stable-60-9592-100-0 | cos-cloud |          |            |   READY
cos-stable-61-9765-79-0 |  cos-cloud | cos-stable |           |  READY

Create VM named urbalurbahost
```
gcloud compute instances create urbalurbahost \
    --image cos-stable-61-9765-79-0 \
    --image-project cos-cloud \
    --zone us-central1-a \
    --machine-type n1-standard-1
```

#### Log in to urbalurbahost from the mac
```
gcloud compute --project "urbalurba-184319" ssh --zone "us-central1-a" "urbalurbahost"
```
#### Configure the urbalurbahost instance 
```
docker-credential-gcr configure-docker
```
ERR: It seems it does not authenticate.Do this instead

```
METADATA=http://metadata.google.internal/computeMetadata/v1
SVC_ACCT=$METADATA/instance/service-accounts/default
ACCESS_TOKEN=$(curl -H 'Metadata-Flavor: Google' $SVC_ACCT/token   | cut -d'"' -f 4)
```
Output is then:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   208  100   208    0     0  69868      0 --:--:-- --:--:-- --:--:--  101k


#### Login to the google docker registry
```
docker login -u _token -p $ACCESS_TOKEN https://gcr.io
```
Login Succeeded


OK. So now we have a VM in google compute engine named urbalurbahost and docker knows about and ca log in to google docker registry where we pushed our container.

### Pull the urbalurba-discourse image to urbalurbahost
```
docker pull gcr.io/urbalurba-184319/urbalurba-discourse:v1
```

### Prepare urbalurbahost host data disk
* Create the dir /home/tec/dockerdisk/urbalurba-discourse/shared
* Create the dir /home/tec/dockerdisk/vmdisk
(tec is my username on urbalurbahost. There is probably a better place to put the data, but I don’t care for now)


### Copy data files from Mac to urbalurbahost

On the Mac host i have a temp dir. Go there
```
cd dockerdisk/vmdisk
```
Copy the files on the Mac that was created on localubuntu to urbalurbahost
``` 
gcloud compute scp standalone_dir.tar urbalurbahost:/home/tec/dockerdisk/vmdisk/standalone_dir.tar
```
Copy and extract the files so that they are ready
```
cp standalone_dir.tar  /home/tec/dockerdisk/urbalurba-discourse/shared
cd /home/tec/dockerdisk/urbalurba-discourse/shared
tar -xvf standalone_dir.tar
rm standalone_dir.tar
```
Yeyy. Now we have the container and the data.


### Open http in the firewall for urbalurbahost
On the https://console.cloud.google.com/compute/instances
Select urbalurbahost and Edit

Firewalls = Allow HTTP traffic

### Run the docker image on urbalurbahost

Now that we have the container on the urbalurbahost it’s time to run it. I copy the docker run with all its parameters from localubuntu. I need to change the following parameters.

* DOCKER_HOST_IP=172.17.0.1  
* The image name : gcr.io/urbalurba-184319/urbalurba-discourse:v1

And the volumes 
* -v /home/tec/dockerdisk/urbalurba-discourse/shared/standalone:/shared 
* -v /home/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log:/var/log 

And I have removed passwords and stuff. Then the docker command looks like this


> docker run -d --restart=always -e LANG=en_US.UTF-8 -e RAILS_ENV=production -e UNICORN_WORKERS=2 -e UNICORN_SIDEKIQS=1 -e RUBY_GLOBAL_METHOD_CACHE_SIZE=131072 -e RUBY_GC_HEAP_GROWTH_MAX_SLOTS=40000 -e RUBY_GC_HEAP_INIT_SLOTS=400000 -e RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=1.5 -e DISCOURSE_DB_SOCKET=/var/run/postgresql -e DISCOURSE_DB_HOST= -e DISCOURSE_DB_PORT= -e DISCOURSE_HOSTNAME=discourse.urbalurba.no -e DISCOURSE_DEVELOPER_EMAILS=notme@urbalurba.no -e DISCOURSE_SMTP_ADDRESS=smtp.elasticemail.com -e DISCOURSE_SMTP_PORT=2525 -e DISCOURSE_SMTP_USER_NAME=notme@urbalurba.no -e DISCOURSE_SMTP_PASSWORD=I-will-not-give-you-this_:)  -h urbadics -e DOCKER_HOST_IP=172.17.0.1 --name urbalurbadiscourse3 -t -p 80:80 -p 443:443 -v /home/tec/dockerdisk/urbalurba-discourse/shared/standalone:/shared -v /home/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log:/var/log gcr.io/urbalurba-184319/urbalurba-discourse:v1 /sbin/boot


Now open a local browser http:// <see external ip address on firewall config above>


# Errors that needs to be solved
But I get a error message from nginx : 502 Bad Gateway
The container is accessing the volumes specified and I see logs for nginx at
/home/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log/nginx

This is where I need help from the community - Do you know why it does not work?


--> sudo cat /home/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log/nginx/access.log
output:
[03/Nov/2017:05:52:17 +0000] "104.198.186.228" 85.165.238.194 "GET / HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "-" 502 316 "-" 0.000 0.000 "-"

--> sudo cat /home/tec/dockerdisk/urbalurba-discourse/shared/standalone/log/var-log/nginx/error.log
output:
2017/11/03 05:52:17 [error] 52#52: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 85.165.238.194, server: _, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:3000/", host: "104.198.186.228"


What is the ngnix log supposed to look like on a functioning setup?
On the Mac the access.log is like this

[03/Nov/2017:06:22:55 +0000] "localhost" 172.17.0.1 "GET / HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "list/latest" 200 8653 "-" 45.054 45.054 "-"
[03/Nov/2017:06:22:59 +0000] "localhost" 172.17.0.1 "GET /images/d-logo-sketch.png HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "-" 304 151 "http://localhost/" - 0.000 "-"

[03/Nov/2017:06:23:25 +0000] "localhost" 172.17.0.1 "POST /message-bus/64222aea49f949239155d33e7590a86a/poll HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "-" 200 719 "http://localhost/" 25.300 25.300 "-"
[03/Nov/2017:06:23:50 +0000] "localhost" 172.17.0.1 "POST /message-bus/64222aea49f949239155d33e7590a86a/poll HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "-" 200 510 "http://localhost/" 25.021 25.021 "-"



 
So trying to get just the image file from urbalurbahost. That works and the log is :
[03/Nov/2017:06:28:29 +0000] "104.198.186.228" 85.165.238.194 "GET /images/d-logo-sketch.png HTTP/1.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/604.3.5 (KHTML, like Gecko) Version/11.0.1 Safari/604.3.5" "-" 200 14672 "-" - 0.000 "-"


