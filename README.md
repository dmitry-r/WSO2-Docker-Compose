# Docker WSO2 Project

## TODO and BUGS

- The first time you call `docker-compose up` or when you call it after you call `docker-compose down`, the mysql container will have to build the database. Sometimes, various wso2 apps will crash because the data provider is not ready in time. `Ctrl C` to kill the process and then running `docker-compose up` again will fix this, because by then the databases have been built. There is a way to have the other containers wait until the mysql container is ready. I get that fix in eventually.

## Docker base images

I have a repo of the dockerfiles at https://github.com/eristoddle/dockerfiles.  That repo can be used to create generic docker images of wso2 applications using a Centos 7 base image. The base image Dockerfile is also included in that repo. The base image should only have to be updated to update Centos. The wso2 images should only have to be updated to install a new version of WSO2.

I used that repo to build the base image and wso2 images and push the images to my Docker Hub account: https://hub.docker.com/u/eristoddle/

## Custom Images

- The ESB image with the features postfix had the HL7 feature built in and the axis2.xml file in this project is configured for that image to activate HL7 functionality. This currently has to be done manually.
- The IS image is custom. I load all the applications up as service providers for SSO and then wrap up the result as an image. This should be able to be done with the sso-idp-config.xml file but it is not parsing the file correctly. Here is the service provider data:

Application  |  Service Provider Name |  Assertion Consumer Service Url
--|---|--
 ESB | service-provider-esb  |  https://localhost:9444/acs
GREG  | service-provider-greg  |  https://localhost:9445/acs
BRS  | service-provider-brs  |  https://localhost:9446/acs
DSS  | service-provider-dss  |  https://localhost:9447/acs
AM  | service-provider-apim  |  https://localhost:9448/acs
AS  | service-provider-as  |  https://localhost:9449/acs

## Getting Started

**For a local dev environment, all you have to worry about is the docker-compose section**

### Running with docker-compose

- Make sure you have docker installed.
- Run `docker-compose up`
- To manage the boxes through a UI on your dev machine, check out http://portainer.io/. Or just run this `docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer` and Portainer will be accessible at http://localhost:9000. I added this to the docker-compose file.
- This brings up 8 boxes running wso2 applications. For Docker on Mac, you will have to set the memory up a little from the default 2gb or one of the  boxes will crash because it ran out of memory. I set mine up to 6gb and it seems to run fine.
- The esb has the business rules server built in and the brs component is just another esb instance.
- The ports for each box can be found in the `docker-compose.yml` file.
- For SSO, you will have to edit the /etc/hosts file on your laptop and add the host `wso2identity` to `127.0.0.1`.

### Local Urls with Docker Compose

Default user: admin
Default password: admin

- ESB: https://localhost:9447/carbon/admin/login.jsp
- BRS: https://localhost:9444/carbon/admin/login.jsp
- GREG: https://localhost:9445/carbon/admin/login.jsp
- DSS: https://localhost:9446/carbon/admin/login.jsp
- IS: https://localhost:9443/carbon/admin/login.jsp
- AM: https://localhost:9448/carbon/admin/login.jsp
- AS: https://localhost:9449/carbon/admin/login.jsp

### When You Don't Need to Run the Whole Stack

Just comment out the images you don't want to run in the docker-compose.yml file. The following you will need to have running due to configuration files looking for them:

- wso2mysql
- is
- greg

## Running with Rancher

NOTE: Not necessary for local development, but for playing with a production level environment locally.

Rancher will currently only run on a Linux host. So installation on Linux is simple. For Mac (and possibly Windows), you will have to use a different process.

- You will still need Docker for Mac installed.
- You will also need Virtualbox.
- You will be using docker-machine commands instead of docker from your actual physical machine.
- You will be using docker commands inside the vm you built for Rancher.

### Creating a Rancher Instance

Most of the steps I learned from this post: https://www.webuildinternet.com/2016/09/06/how-to-install-rancheros-and-rancher/.

- Run `docker-machine create -d virtualbox --virtualbox-boot2docker-url https://releases.rancher.com/os/latest/rancheros.iso rancheros` to create the docker host for rancheros
- Log in to the machine with `docker-machine ssh rancheros`
- Then run `sudo docker run -d --restart=always -p 8080:8080 rancher/server` inside this new host to start rancher.

### Managing the Rancher Instance

- The first thing you need to do after following the steps in that post, is create a host after you browse to the Rancher web address.
- I used the 192.168.99.100 address that was already in first form for adding hosts.
- On the second form, I did nothing other than run the command listed at the bottom in the newly created vm. If you followed the instructions in the post above and you're Rancher vm is named `rancheros`, then you would run `docker-machine ssh rancheros` to get shell access to the box and then run the command from the Rancher UI there.
- Click the Close button at the bottom and your new host should show up in a few seconds.
- I found that starting the vm with 6gb of ram instead of the default 1gb makes things come up quickly. I tried 4gb and it came up but you had to wait longer. To change the vm's settings, first run `docker-machine stop rancheros`. Then run Virtualbox and find the vm and change it's setting to have more memory. I also added 2 cores from the processor instead of 1. Then you can restart Rancher again with `docker-machine start rancheros`.

### Other Rancher/Docker commands & quirks

Because stuff happens.

- To restart docker running in RancherOS: `sudo ros service restart`
- Stop all containers: `docker stop $(docker ps -a -q)`
- Remove all containers: `docker rm $(docker ps -a -q)`

## Volumes in Environments Other Than Local

Locally we can sync the volumes to our development machine. On remote hosts, we have to wrap the configuration files into container that will sync the files to the host machine for the other containers to use. That is the reason for the Dockerfile in this project and the difference between the various docker-compose files. The standard docker-compose.yml file is for local development.

This means the image created from this docker file must be pushed to a private repo and tagged. Then referenced in the docker compose yaml file for that environment.

This is an old method for doing this and probably should be changed to use volume drivers like:
- Flocker
- Convoy
- NFS
  After some research on the what would work best for us.

## More Docker Reading

- https://github.com/veggiemonk/awesome-docker
- https://www.katacoda.com/

## WSO2 Development

### Governance Registry Persistance

The [Governance Registry](http://wso2.com/products/governance-registry/) is used to provide a shared governance partition backed by a MySQL database, as documented [here](https://docs.wso2.com/display/ESB490/Governance+Partition+in+a+Remote+Registry). The database `registrydb` is created by the `scripts/mysql/greg-init.sql` script on-start.

To test the shared governance partition set-up, navigate to the `/_system/governance` registry from any of the web consoles. Add or modify some resources, and expect the changes to be seen in the web consoles of other components. Note that caching is disabled in the `registry.xml` file of each component.

There are two others adjustments I had to make to get this to work:

1. Override the default MySQL `sql-mode` using the `conf/mysql/my.cnf` script to remove the [`NO_ZERO_IN_DATE`](http://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_zero_in_date) and [`NO_ZERO_DATE`](http://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_zero_date) restrictions. WSO2 uses `DEFAULT 0` in some of their timestamp queries.
2. Disable SSL by setting the `useSSL` parameter in the JDBC connection string as seen in the `conf//master-datasources.xml` scripts.

### Single Sign-On

The [Identity Server](http://wso2.com/products/identity-server/) is configured to support web browser-based SSO across all the components based on the steps described [here](https://docs.wso2.com/display/IS510/Configuring+SAML2+Single-Sign-On+Across+Different+WSO2+Products). A MySQL database is used as the [backing data source to store registry and user manager data](https://docs.wso2.com/display/IS510/Setting+up+MySQL). The database `identitydb` is created by the `scripts/mysql/is-init.sql` script on-start.

Instead of defining the service provider for each component via the administrator console, I specified them in the `sso-idp-config.xml` file in accordance to this [example](https://docs.wso2.com/display/IS510/Configuring+a+SP+and+IdP+Using+Configuration+Files). There is an issue with logout where the Identity Server throws an `ERROR {org.wso2.carbon.identity.sso.saml.processors.LogoutRequestProcessor} - No Established Sessions corresponding to Session Indexes provided.` exception.

Since I am using Docker machine, I have to add the Identity Server hostname (`wso2identity`) to my `/etc/hosts` file. Refer to [Usage](https://github.com/ihcsim/compose-wso2#usage) section on the updates necessary for the `/etc/hosts` file. Otherwise, by default, all the Identity Server SSO web applications will redirect SAML requests back to `localhost`.

*NOTE*: I still had to login to the Identity Server and add the service provider there. For some reason I have yet to get the sso-idp-config.xml file in the wso2is conf to load these on launch. See details [here](https://docs.wso2.com/display/IS500/Enabling+SSO+for+WSO2+Servers).

I have created an image which has these service providers set up already, but it's a manually built image. I plan on scripting this by populating the correct tables in the database.

### Port Offsets

I am using port offsets in the carbon.xml files to prevent ports from colliding internally when doing SSO. You will also notice this in the docker-compose.yml file.

### Connecting the ESB and DSS

The DSS image has drivers preinstalled for Mysql, Sql Server and Postgres.

SEE:
- http://kalpads.blogspot.com/2012/07/implement-mdm-pattern-using-wso2-esb.html
- http://chathurikaerandi.blogspot.com/2016/05/how-do-i-integrate-wso2-esb-and-wso2.html

### Creating an API in the ESB

SEE:
- https://docs.wso2.com/display/ESB490/Creating+APIs
- http://wso2.com/library/articles/2012/09/get-cup-coffee-wso2-way/

### Using the Governance Registry

SEE:
- http://wso2.com/library/articles/2015/07/article-multi-environment-artifact-management-for-wso2-products-using-wso2-governance-registry/

### Using the Business Rules Server

SEE:
- http://wso2.com/library/articles/2011/04/integrate-rules-soa/
- http://wso2.com/library/articles/2013/05/eclipse-plugin-wso2-business-rules-server/

## EC2 Deployment

For ECS deployment, I wrapped up the configuration files into new docker images, since it seems that AWS only allows you to sync folders and not single files. The other option was syncing whole folders and the repository folder is not that small. And since these files should not have to be touched that often after initial configuration, it seemed the best route.

Running `./build_push_all.sh` will build all docker images for deployment and push them to docker hub except for:

- ESB
- IS

which I currently custom build to include HL7 and SSO. I am working on an automated build for those 2.

### Using ecs-cli

For this use the `docker-compose-aws-dev.yml` file.

SEE: https://github.com/aws/amazon-ecs-cli

#### Running

Running an instance with `ecs-cli compose` has its limitations. Since you are using a docker-compose file, you can't set the `confdata` container's `essential` key to false or set the memory limit on the containers even though there is a `memory_limit` key you can add to the compose file. So the containers come up and then go back down since the default memory limit is 512 and essential is set true by default.

I got around this by launching the instance with `ecs-cli compose` and then going to the aws website and modifying the task definition to up the memory and set the confdata container to non-essential and relaunching, so going forward it will be better to use `aws cli` and the task definition json file so these settings can be used. Giving the wso2 containers 1gb memory each allowed them to run.

You will also have to set up a security group for all the ports you need open to the outside world or modify the one that gets created when launched.

1. Configure the cli:

```
ecs-cli configure --region us-west-2 --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY --cluster bardavon-wso2-cluster
```

2. Create a cluster:

```
ecs-cli up --keypair id_rsa --capability-iam --size 1 --instance-type t2.medium
```

3. Bring up the containers:

```
ecs-cli compose --file docker-compose-aws-dev.yml up
```

#### Stopping

I kept killing my instance and it would only respawn. I even delete the cluster and task definitions from the web interface and the instance would not die. The important step was setting the cluster size to zero.

1. Scale down the cluster:

```
ecs-cli up --keypair id_rsa --capability-iam --size 0 --instance-type t2.medium
```

2. Bring the cluster down

```
ecs-cli down --force
```
