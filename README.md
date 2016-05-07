## General Setup

We are using Rancher to manage our docker hosts and containers on AWS.
Rancher has a web UI to inspect and configure the setup.
The docker hosts are running RancherOS, which can be configured using a cloud-init structure
passed through the userdata when AWS EC2 starts up an instance.

LARA is autoscaled using AWS EC2 autoscaling. When the autoscaller launches a new host, the new
host contacts rancher and rancher then adds all of the configured docker containers to the host.

## Updating Code ##

A new lara release should be tagged. The typical way to do this is to use GitHub's release UI.
The new tag will be automatically built by dockerhub. Once the release has been built,
go to rancher: rancher-devops.concord.org to continue the release.

Within rancher select the 'app' service, and then uses the menu to select upgrade.
On the upgrade form you can specify the docker image. Set this to be the new tag that dockerhub
built for you.

If you want a seamless upgrade, select: "Start new containers before stopping old".

After the upgrade you need to "Finish the Upgrade", which is in the same menu. This deletes the old
containers that rancher keeps around incase you want to roll back the upgrade.

If you have migrations to run you need to do that manuall by opening a rails console on one of the app
containers. See below for more details.

Finally you should also upgrade the worker service with your new image.

## Open a rails console from the command line:

For simple stuff you can use the Rancher UI. It has an "Open Console" in the menu of containers.
Copy and paste is supported using the mouse. Keyboard copy and paste uses non standard keys.

For more complex console work you probably want a native console instead:

- `ssh -i [devops.pem location] rancher@[public hostname of a lara-docker server]`
- `docker ps` (to find the name of the containers)
- `docker exec -it r-lara-staging_app_1 bash` (note the name of app container might be different)
- `rails c`

With a little work it seems we could setup a tool that would make this easier. It could query AWS to automate the
steps above.  Or it could use the node library term.js-cli to connect to the rancher server console API.
Ideally a developer would just run something like `rancher lara-staging run rails console`.
That might be possible by combining rancher-compose with the rancher api and term.js-cli.
Or possibly by using rancher-compose, the rancher api, and ec2 api.
It would be worth discussing this with the rancher devs.

## Work with rancher server on commandline:

This is probably not needed often. I used it once to be able to run the mysql client
on the rancher server because that is a good place that has access to the mysql database.

- ssh into rancher-server instance (it is rancheros so user is rancher)
- run `docker ps` to find the name of the containers
- run `docker exec -it [rancher container name] bash`

If you do need to connect with mysql you'll need to look up the ENV variables that were used
when the rancher container was launched.

## New relic server monitoring

Currently this is setup inside of the lara stack in rancher. The server monitor is a system level service that mounts
several folders. However the drive mounting and the /proc mounting was not easy to setup at the time it was setup
so some metrics are not correct about the server. However by having it setup, it does at least let NR know we are using
docker which shows up in the UI as a server with multiple containers running on it.

## Logs Sent to AWS CloudWatch

Part of the configuartion for the App service and Worker service in Rancher defines the AWS log driver.
It also specifies which log string the service should send its logs too.

The aws log driver needs AWS credentials to send the logs to CloudWatch. These credentials need to be available
to the docker daemon itself. To handle this we are using a EC2 instance role, which is specified in the launch
configuration. This is a feature of EC2 and IAM. Dynamic credentials are created by IAM and made available to the
EC2 instance using a metadata URL.  The Docker AWS Log driver has built in support for checking this
metadata URL.

Currently the app service logs to one logstream and the worker service logs to a different logstream.

## AWS Autoscaling setup

Here is the current userdata with senstive info removed:

```
#cloud-config
rancher:
  services:
    rancher-agent1:
      image: rancher/agent:v0.11.0
      command: [rancher url specific for environment]
      privileged: true
      volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      environment:
        CATTLE_HOST_LABELS: application=lara-staging
```

You can see this UserData in the lara-cloudformation-stack.json file.

The rancher-agent parts of this cloud-config were based on the information provided here:
http://docs.rancher.com/os/running-rancher-on-rancheros/

## Load Balancing ##

There are 2 levels of load balancing in our setup. The AWS load balancer is hit first by a client. It
then sends the request on to one of the hosts (EC2 instances) running the LARA rancher stack. Each of
these hosts is running the Rancher load balancer. This rancher load balancer distributes the request
to any container running the app service. So a rancher load balancer on host A might send the request
to an app container on host B.

The AWS load balancer healthcheck is configured to check a special path. This path is just a rancher
load balancer status path. So the AWS load balancer is not checking the health of the app instances
themselves. It is just checking if the rancher load balancer on each host is running.

Rancher has its own healthchecks. For the app service these are configured to check the index of LARA.
If rancher finds a container is failing the healthcheck then it will restart the container.

The AWS Autoscaler does not use the AWS loadbalancer's healthchecks. This removes at little bit of
complexity when trying to figure out why the autoscaler decided to shutdown a instance.

### Reaping host definitions from rancher

When a host is shutdown in EC2 (probably by the AWS auto scaler) the host definition
is removed from rancher. We call this 'reaping'.

This is handled by a 2 step approach:

1. the lara rancher stack is configured run our custom rancher-host-manager image one time. This image
looks up the EC2 meta data and updates the meta data in rancher itself to include some of this
EC2 metadata. Most importantly it sets the name of the host in rancher to be the EC2 instance-id.
The code for this image is here: https://github.com/concord-consortium/rancher-host-manager

2. there is a seperate docker container running in the 'rancher-utils' environment in rancher.
This container wakes up ever few seconds and checks if there are any any 'reconnecting' hosts
in rancher. If it finds one then it checks to see if the host exists in EC2. If the host doesn't
exist then it removes the host from rancher. The source for this container is in the same repository:
https://github.com/concord-consortium/rancher-host-manager

#### Notes about Preprovisioned AMI

I tried using a preprovisioned AMI inorder to speed up the launching of a new server into the autoscaling group. However
it didn't seem make that big of a difference. With a new version of the LARA docker image the server was still taking
11 minutes to complete its startup.

Incase we want to go back to the pre-provisioned approach here are some notes.

The ami was created by:
- launching an instance with the rancheros-0.4.2 ami
- using the same userdata as above
- waiting for it to show up in the rancher UI and finish initializing
- removed it from the rancher UI (this seems to stop all of the rancher managed containers)
- the rancher state was removed based on these instructions:
    http://docs.rancher.com/rancher/faqs/agents/#using-a-cloned-vm
- the existing containers were stopped and removed, otherwise they conflicted with the containers rancher
  sets up.  This is done with `docker rm -v $(docker ps -a -q)` (you might need to kill the containers first)
- the cloud-init system containers were removed which seems to be necessary to clear out any cloud-init configuration
  from the previous launch:
`sudo system-docker rm -v cloud-init-pre; sudo system-docker rm -v cloud-init`. The cloud-init container seemes to save the
the configuration that is passed in through the EC2 userdata, so on restart this configuration is loaded. Which means the
instance will join the rancher environment used to create the AMI.
- the instance was shutdown using `shutdown -h now`, and then imaged in the AWS console

It seems most of the provisioning time is spent downloading the docker images from dockerhub. A better alternative
is for us to run our own docker caching repository. Then the downloading should be very quick in most cases.

#### Notes about issue with hostname in rancher

When RancherOS is started, the 'name' for a host displayed in rancher is just 'rancher'.
Originally this was not a problem, and now it seems always be a problem.
At first it seemed that the 0.4.3 upgrade caused the problem because it started at the same time. But going back to 0.4.2
RancherOS instance didn't seem to solve the problem. However we should verify that if we are going to submit a bug report.

If I understand how things are configured, this hostname should be set when cloud-init starts up on RancherOS. It should query the EC2 meta-data service to find the hostname.

I tried adding a label to the rancher-agent config in the userdata to make sure it started after
cloud-init set the host name but that didn't help. Here is the label `io.rancher.os.after: cloud-init`

When using the pre-built ami described above, the hostname is that of the of the original server.

When I upgraded rancher, it restarted the agents and that updated the host name. However in that case when I ssh'd into the hosts I believe the correct hostname was shown. Now in somecase sshing into the host still shows the wrong hostname.

Another clue is that when using `system-docker inspect` on any of the stopped or running containers they all also list this wrong hostname. So the problem might be that the containers get this hostname when they were are initialy launched.

In the AMI creation steps above the cloud-init container is removed before making the AMI. The hope was this would force cloud-init to re-run and then fix the problem. It did not. The rancher os startup.yml file is what initially launches all of these services so perhaps there is someway to tweak it that will fix the problem.

## Thing still to improve

### Adding Version info to LARA webpages

Currently the docker app doesn't show any version info.

There are a couple of ways to go about this:
- get the git branch or tag into the image, and then have the web app render it for users
- get access to the image tag that the current container was created from, and show that to the user

Inorder to put the git branch or tag in the image it requires the system building the image to
provide this information to the process doing the building. DockerHub builds do not provide this info.
If the image is built and pushed from Travis then the information is provided. So using Travis seems
the best option for taking this approach.
One way to do this is for the Dockerfile to declare a build argument. And then in the docker build 
command run by Travis it would would pass the Git branch to this argument. The dockerfile can provide
it to the image as an environment variable. Or it could run a command to inject it into the version.yml
file that is already supported by LARA.

Inorder to get the image tag that the container was started from. This approach could be used:
- the lara app queries the meta-data service to get the container uuid.
- then it quieries the rancher API to get find the matching container and then get its imageUuid
- the imageUuid shows the image name and tag

It might be best to run this in an initializer at the startup of the app, however it could slow down the
loading of the app. Another approach is to only run this from a second info page in the UI. This second
page could even provide links into rancher UI for the stack and the service.

One danger with using the rancher API from the app container is that it opens a security hole. We can provide
readonly access to rancher API. But in the results rom the API are all of the secret keys for the app.
Providing access to those generally to the app doesn't seem like a good idea.

### Pull nginx out of lara container

A separate nginx container can be used that proxies all requests to LARA.  And it would be setup to
agressively cache the assests from the LARA container. Benefits:
- we can drop Foreman.
- the logging would be more clear: standard out from nginx will go to one stream, and lara to another stream
- starting or upgrading a new server might be faster because the lara container wouldn't need to contain nginx
too.

## Other Annoying issues:

- during local testing, the virtualbox machine setup by docker-machine seems to loose its
internet connection sometimes. This is the same thing I've experience with Virtual Box.
I'm sure not under what conditions it happens.

- the delayed job worker logs info each time it wakes up and looks for new jobs, so this makes the default log full of
mostly pointless messages.

