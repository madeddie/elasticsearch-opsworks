# DEPRECATED

This module hasn't been maintained nor used by me for some time. Security warnings have started popping up.

Use only as inspiration or starting point for your own projects.

[![No Maintenance Intended](http://unmaintained.tech/badge.svg)](http://unmaintained.tech/)

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

    rake provision AWS_ACCESS_KEY_ID=<aws key> AWS_SECRET_ACCESS_KEY=<aws secret> STACK_NAME=edwin

Open `https://<environment>.<route53 zone name>/_plugin/head`

Destroy the environment:

    rake destroy AWS_ACCESS_KEY_ID=<aws key> AWS_SECRET_ACCESS_KEY=<aws secret> STACK_NAME=edwin


## Infrastructure details

    Route53 --> ELB --> EC2 attached to EBS volumes

* Index will be stored on EBS volumes, mounted at `/mnt/elasticsearch-data`
* One master node by default, 2-node cluster by default
* Load balanced by an ELB
* ELB listens on HTTP/HTTPS, configured with basic auth challenge (nginx)
* ELB and instances listen on 9200 and 9300 without auth to members of their respective security groups
* EC2 instance type defaults to `c3.large`

## Variables

These variables are available to be given or override the default:

| Variable                         | Default                   | Explanation                        |
|----------------------------------|---------------------------|------------------------------------|
| AWS_ACCESS_KEY_ID                |                           | needed, will not run without       |
| AWS_SECRET_ACCESS_KEY            |                           | needed, will not run without       |
| AWS_REGION                       | eu-west-1                 |                                    |
| STACK_NAME                       | my                        | name of the CF and OpsWorks stacks |
| VPC                              | devvpc                    |                                    |
| SUBNETS                          | SubnetData1A,SubnetData1B |                                    |
| INSTANCE_COUNT                   | 2                         |                                    |
| SSL_CERTIFICATE_NAME             | wild.appdev.io            |                                    |
| SSH_KEY_NAME                     | dev@lgi                   |                                    |
| ROUTE53_ZONE_NAME                | appdev.io                 |                                    |
| SEARCH_DOMAIN_NAME               | $STACK_NAME.$ZONE_NAME    | i.e. `my.appdev.io`                |
| SEARCH_USER                      | elasticsearch             |                                    |
| SEARCH_PASSWORD                  | password                  |                                    |
| INSTANCE_TYPE                    | c3.large                  |                                    |
| ELASTICSEARCH_VERSION            | 1.6.0                     |                                    |
| ELASTICSEARCH_AWS_PLUGIN_VERSION | 2.6.0                     | needs to be compatible to ES version |
| SKIP_INSTANCE_PACKAGE_UPDATES    | false                     | if true doesn't install updates on provision |
| SKIP_DELETE_VOLUMES              | false                     | if true doesn't delete EBS volumes on destroy |
| REPLACE_INSTANCES                | false                     | if true replaces existing instances on re-provision |
