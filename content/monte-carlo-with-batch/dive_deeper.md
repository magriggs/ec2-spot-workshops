---
title: "Dive Deeper: the underlying risk compute environment"
date: 2021-09-06T08:51:33Z
weight: 55
hidden: True
---

## Dive Deeper
This optional section will give you more knowledge of how the underlying technology is used to calculate our risk metrics. 

## Spot Instance Draining
Note that the `UserData` of the created Launch Template contains the following script:

```
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="==MYBOUNDARY=="

--==MYBOUNDARY==
Content-Type: text/x-shellscript; charset="us-ascii"

#!/bin/bash
echo "ECS_CLUSTER=EcsSpotWorkshop" >> /etc/ecs/ecs.config
echo "ECS_ENABLE_SPOT_INSTANCE_DRAINING=true" >> /etc/ecs/ecs.config
echo "ECS_CONTAINER_STOP_TIMEOUT=90s" >> /etc/ecs/ecs.config
echo "ECS_ENABLE_CONTAINER_METADATA=true" >> /etc/ecs/ecs.config

--==MYBOUNDARY==--
```

What we are doing here is enabling [Spot Instance Draining](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-spot.html). When ECS Spot Instance draining is enabled on the instance, ECS receives the Spot Instance interruption notice and places the instance in DRAINING status. When a container instance is set to DRAINING, Amazon ECS prevents new tasks from being scheduled for placement on the container instance. To learn more about Spot instance interruption notices, visit [Spot Instance interruption notices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html#spot-instance-termination-notices).
For more information on how AWS Batch uses Amazon ECS, see the [FAQs](https://aws.amazon.com/batch/faqs/).

### Dockerfile
The Dockerfile contains a set of instructions that Docker executes to build up a container image. If you open the [Dockerfile](https://raw.githubusercontent.com/magriggs/ec2-spot-workshops/master/content/monte-carlo-with-batch/monte-carlo-with-batch.files/docker-files/Dockerfile) you will see that it starts from a base image that has Python pre-installed. It then installs some utility applications, like curl, zip, and jq. The AWS CLI is installed, and then curl is used to download the ``Autocallable.Note.py`` file from Github. Dockerfile creates a Python environment, and then installs the Python library dependencies listed in ``requirements.txt``, such as QuantLib, numpy, and scipy. Finally, it modifies the ``montecarlo-price.sh`` script to be executable, and sets the script as the ENTRYPOINT for the container.

### montecarlo-price.sh
``montecarlo-price.sh`` is the ENTRYPOINT for the Docker container, and is executed when AWS Batch calls ``docker run`` on the container. As this script is invoked by the AWs Batch job, it receives a set of environment variables and input parameters that it uses to extract trade parameter data from S3, using S3 Select, and pass that on to ``Autocallable.Note.py``, which executes the actual risk calculation. The results are then uploaded to an output bucket in S3, which will be picked up by the final stage of the Step Functions pipeline, and merged into the final portfolio risk results.

### Autocallable.Note.py
Mikael Katajamaki CQF, kindly made public his code for valuing an autocallable using QuantLib. The market data such as spot price and volsurface are defined inside the Python script, so whilst the results from the script are not accurate for production use, they are a good example of how to use QuantLib without getting caught up in the details of connecting to market data sources, and so on. A detailed discussion of the mechanics of the Autocallable product, and the code that implements them, can be read on Mikael's [blog](http://mikejuniperhill.blogspot.com/2019/11/quantlib-python-heston-monte-carlo.html). 

