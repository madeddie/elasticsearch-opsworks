# ElasticSearch OpsWorks

Copied and adapted from [this repo] (https://github.com/ThoughtWorksStudios/elasticsearch-opsworks)

## Before deployment

Please setup the following dependencies in your AWS region:

* VPC, defaults to `devvpc`
* Subnet(s), defaults to `SubnetData1A,SubnetData1B`
* SSL certificate, defaults to `wild.appdev.io`
* An SSH key pair, defaults to a keypair named `dev@lgi`
* A Route53 zone for the domain, defaults to `appdev.io.`
* The default `aws-opsworks-service-role` and `aws-opsworks-ec2-role` need to exist before provisioning. OpsWorks should automatically create these roles when you add your first stack through the OpsWorks console. See http://docs.aws.amazon.com/opsworks/latest/userguide/gettingstarted-simple-stack.html and http://docs.aws.amazon.com/opsworks/latest/userguide/opsworks-security-appsrole.html for details.

## Setup environment

* Clone this repository
* Run `bundle install` to install the necessary gems

## Usage

Provision the environment:

    rake provision AWS_ACCESS_KEY_ID=<aws key> AWS_SECRET_ACCESS_KEY=<aws secret> AWS_REGION=eu-west-1 ENVIRONMENT=edwin

Open `https://<environment>-es.<route53 zone name>/_plugin/head`

Destroy the environment:

    rake destroy AWS_ACCESS_KEY_ID=<aws key> AWS_SECRET_ACCESS_KEY=<aws secret> AWS_REGION=eu-west-1 ENVIRONMENT=edwin


## Infrastructure details

    Route53 --> ELB --> EC2 attached to EBS volumes

* Index will be stored on EBS volumes, mounted at `/mnt/elasticsearch-data`
* One master node by default, 2-node cluster by default
* Load balanced by an ELB
* ELB listens on HTTP/HTTPS, configured with basic auth challenge (nginx)
* Instances listen on 22, 9200 and 9300 without auth
* EC2 instance type defaults to `c3.large`

## Variables

These variables are available to be given or override the [default]:

* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY
* AWS_REGION
* VPC [devvpc]
* SUBNETS [SubnetData1A,SubnetApps1B]
* ENVIRONMENT [my]
* ROUTE53_ZONE_NAME [appdev.io]
* INSTANCE_COUNT [2]
* SSL_CERTIFICATE_NAME [wild.appdev.io]
* SEARCH_DOMAIN_NAME [$ENVIRONMENT-es.$ROUTE53_ZONE_NAME] (so, if all are not gives `my-es.appdev.io`)
* SSH_KEY_NAME [dev@lgi]
* SEARCH_USER [elasticsearch]
* SEARCH_PASSWORD [password]
* ELASTICSEARCH_VERSION [1.6.0]
* ELASTICSEARCH_AWS_PLUGIN_VERSION [2.6.0]
* SKIP_INSTANCE_PACKAGE_UPDATES [false]
* SKIP_DELETE_VOLUMES [false]
* INSTANCE_TYPE [c3.large]
* REPLACE_INSTANCES [false]
