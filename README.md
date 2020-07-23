# sxselenium

This repository contains files needed to setup sxselenium: a selenium grid
server with [warsaw](https://www.dieboldnixdorf.com.br/warsaw) installed.

To try it locally just `docker-compose up -d`.

## how it works

There are two needed applications to run selenium grid it in production:
selenenium-hub and selenium node

## aws elastic beanstalk application deployment

To deploy it, zip it's files (in the main 'folder' of the archive) and deploy to elastic beanstalk.
Go to `All Applications -> sxselenium -> Application versions` and upload the zip file there

**TODO**: we need to implement an auto-deploy function.

For now, don't forget to follow the name convention (`sxselenium-manual-<version-number>`) and to create the corresponding git tag in this repo!


## aws elastic beanstalk instances creation

To create the elastic-beanstalk instances through the aws web interface. **Remember: After creating nodes cluster it is necessary to reboot it in order to work properly.**

**TODO**: we need to translate this into an `eb-cli` script

### security group creation
	Go to `EC2 > Security Group` and create a group named `external-access` with the following rules:

	Inbound:
	Type HTTP, Protocol TCP, Port Range 80, Source 0.0.0.0/0
	Type SSH, Protocol TCP, Port Range 22, Source 0.0.0.0/0

	Outbound:
	Type All Trafic, Protocol All, Port Range All, Destination: 0.0.0.0/0


### configurations for both app types (hub/node):

	- create environment environment -> web server environment

		- name (hub/nodes)
		- plataform: Docker
		- Existing version -> the last app version you uploaded

		- configure more options (do not click `create` yet!)

			- software
				- Server proxy: None

			- instancia
				- t3.small
				- security groups
					default
					external-access

			- rede
				- vpc default
				- marcar duas zonas comuns pros dois componentes

			- security
				ec2 key pair - pra gente poder acessar ela


### configurations for hub app

	- software
		- vars: `SE_OPTS=-role hub -debug -newSessionWaitTimeout 5000 -timeout 600` [1] [2]

	- capacity:
		- Single Instance (default)

### configurations for node

	- software
		- ENV `SE_OPTS=-role node -hub http://sxselenium-hub.h8ujdmcxpw.us-east-1.elasticbeanstalk.com:80/grid/register -debug`
		- ENV `AUTO_GET_IP=true`
		- ENV `NODE_MAX_INSTANCES=5`
		- ENV `NODE_MAX_SESSION=5`

	- capacity:
		- load balanced
		- Instances: 1..4 [3]
		- autoscale (scaling parameters doesn't matter for now [4])


### some details

[1] `newSessionWaitTimeout 5000` : 5 seconds until giving up waitin a node to become available to serve a request, when the grid is full

[2] `timeout 600` : 10 minutes to release an inactive browser session

[3] This minimum nodes value is our manual scaling control while we don't implement an auto-scaling approach that works (see [4]).

[4] Every other capacity parameter intended to autoscale nodes will not work out of the box with selenium grid, once the hub stops sending requests to full nodes, causing it to never trigger any configurable condition.

**TODO:** We'll have to impement a smarter approach for this, maybe using a script in the hub node that uses selenium grid api for polling and scaling.


## Managing environment

web interfaces for monitoring selenium grid
	`http://<hub-eb-host>/grid/console` - here you can see nodes and browsers status
	`http://<node-ec2-instance-host>/wd/hub` - here you can see single node status

ssh instances:
- `ssh -i ~/.ssh/key-bxblue.priv ec2-user@<host>`
	(ask your teamates for the private key)
	(<host> must be the ec2 instance for the nodes, because the eb domain name is the load balancer)
