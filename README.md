# Turbine [![sync]][ci]

[sync]:
https://img.shields.io/badge/CFN-deploy-green.svg?style=flat-square&logo=amazon
[ci]: #get-it-working

Turbine is the set of bare metals behind a simple yet complete and efficient
Airflow setup. Deploy in a few clicks, configure in a few commands, personalize
in a few fields.

The project is intended to be easily deployed, making it great for testing,
demos and showcasing Airflow solutions. It is also expected to be easily
tinkered with, allowing it to be used in real production environments with
little extra effort.

## Overview

![Designer](https://raw.githubusercontent.com/villasv/turbine/master/aws/cloud-formation-designer.png)

The stack is composed of two main EC2 machines, one for the Airflow UI and one
for the job scheduler. Airflow worker machines are instantiated on demand when
the job queue average length grows past a certain threshold and terminated when
the queue stays empty for too long.

Supporting resources include a RDS instance to host the Airflow metadata
database, a SQS instance to be used as broker backend, an EFS instance to serve
as shared configuration and auto-scaling triggers for workers. All other
resources are the usual boilerplate to keep the wind flowing.

## Get It Working

### 0. Prerequisites

- [A key file for remote SSH access][awsdocs-keys]

[awsdocs-keys]:
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html


### 1. Deploy the Cloud Formation Stack

Create a new stack using the latest template definition at
[`aws\cloud-formation-template.yml`][raw-template]. The following button will
readily deploy the stack (defaults to your last used region):

[raw-template]:
https://s3.amazonaws.com/villasv/turbine/aws/cloud-formation-template.yml

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?templateURL=https://s3.amazonaws.com/villasv/turbine/aws/cloud-formation-template.yml)

The stack resources take around 10 minutes to create, while the airflow
installation and bootstrap another 3 to 5 minutes. After that you can
already access the Airflow UI and setup your own Airflow DAGs.

### 2. Setup your Airflow files

This can be done in several ways. What really matters is:

- The Airflow home directory should be accessible at `/efs/airflow`
- The DAG files directory should be accessible at `/efs/dags`

The home directory is assumed to contain the Airflow configuration file
(`airflow.cfg`). This is flexible enough to accommodate pretty much any project
structure and is easily set up with symbolic links.

The usual procedure goes as follows: SSH into the `turbine-scheduler` EC2
instance, clone your Airflow files **inside the shared directory** (`/efs`),
install your stuff and link your specific files.

```
ssh -i "your_key.pem" ec2-user@xxx.xxx.xxx.xxx
cd /efs
git clone https://your.git/user/repo
sudo pip install -r /efs/repo/requirements.txt
sudo ln -s /efs/repo/airflow/home /efs/airflow
sudo ln -s /efs/repo/airflow/dags /efs/dags
```

> **GOTCHA**: configuring the CeleryExecutor also requires listing your region
> as a broker transport option to be passed to Celery, until kombu is configured
> to pick it up automatically (https://github.com/celery/kombu/issues/950).
>
> ```
> [core]
> executor = CeleryExecutor
> ...
> [celery_broker_transport_options]
> region = us-east-2
> visibility_timeout = 21600
> ```

### 3. Clean Up

In order to get a healthy initial bootstrap for your Airflow setup, it's best to
make sure we're writing on a blank slate. The stack comes with Airflow already
running by default because it's easier to demo and to setup supervision.

One way to be completely safe is to stop all airflow process (webserver,
scheduler and workers), reset the airflow database, empty the queue and resume
all airflow processes back again. If you have many workers, the easiest option
is to downscale beforehand.

## FAQ

1. Why is there a `Dummy` subnet in the VPC?

    There's no official support on CloudFormation for choosing in which VPC a
    RDS Instance is deployed. The only alternatives are to let it live in the
    default VPC and communicate with peering or to use DBSubnetGroup, which
    requires associated subnets that cover at least 2 Availability Zones.

## Contributing

>This project aims to be constantly evolving with up to date tooling and newer
>AWS features, as well as improving its design qualities and maintainability.
>Requests for Enhancement should be abundant and anyone is welcome to pick them
>up.
>
>Stacks can get quite opinionated. If you have a divergent fork, you may open a
>Request for Comments and we will index it. Hopefully this will help to build a
>diverse set of possible deployment models for various production needs.

See the [contribution guidelines](/CONTRIBUTING.md) for details.

You may also want to take a look at the [Citizen Code of
Conduct](/CODE_OF_CONDUCT.md).

## Licensing

> MIT License
>
> Copyright (c) 2017 Victor Villas

See the [license file](/LICENSE) for details.
