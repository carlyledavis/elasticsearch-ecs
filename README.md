# elasticsearch-ecs
An elastic search docker container intended to be hosted on ecs.  Forked from http://blog.dmcquay.com/devops/2015/09/12/running-elasticsearch-on-aws-ecs.html


### Installed Plugins

##### Elasticsearch-head

A Basic Frontend for the elasticsearch cluster  
https://github.com/mobz/elasticsearch-head


##### aws-cloud

The official elasticsearch plugin for discovery in aws  
https://www.elastic.co/guide/en/elasticsearch/plugins/current/cloud-aws.html


### Recommended EC2 Permissions

Because the container relies on EC2 discovery to form a cluster, EC2 instances spun up in the ECS cluster will need to be able to make calls to the EC2 service. If your current ECS Instance IAM Role currently isn't granted ec2:DescribeInstances action, you might want to create a new IAM Policy with it.

```js
{
    "Statement": [
        {
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Effect": "Allow",
            "Resource": [
                "*"
            ]
        }
    ],
    "Version": "2012-10-17"
}
```


### Mounting a Volume and Making it Available to ECS

The following userdata can be used to mount an EBS volume labelled /dev/sdb to /opt/vol00 on the EC2 instance - which can then be referenced in the task defenition.

```bash
#!/bin/bash
echo ECS_CLUSTER=Elasticsearch >> /etc/ecs/ecs.config

# Mount /dev/sdb EBS volume to /opt/vol00
mkfs -t ext4 /dev/sdb
mkdir /opt/vol00
mount /dev/sdb /opt/vol00
echo "/dev/sdb /opt/vol00 ext4 defaults,nofail 0 2" >> /etc/fstab

# This prevents docker from throwing a "device-mapper: thin: Deletion of thin device 1 failed."  exception when restarting, more investigation is needed to determine why
docker volume ls

# The Docker daemon must be restarted to see the new mount
sudo service docker restart
```


### Sample Task Definition

items marked with {{ }} change between aws accounts, so aren't listed here.

```js
{
  "requiresAttributes": [],
  "taskDefinitionArn": "{{ task definition arn }}",
  "status": "ACTIVE",
  "revision": {{ task definition revision }},
  "containerDefinitions": [
    {
      "volumesFrom": [],
      "memory": 1500,
      "extraHosts": null,
      "dnsServers": null,
      "disableNetworking": null,
      "dnsSearchDomains": null,
      "portMappings": [
        {
          "hostPort": 9200,
          "containerPort": 9200,
          "protocol": "tcp"
        },
        {
          "hostPort": 9300,
          "containerPort": 9300,
          "protocol": "tcp"
        }
      ],
      "hostname": null,
      "essential": true,
      "entryPoint": null,
      "mountPoints": [
        {
          "containerPath": "/usr/share/elasticsearch/data",
          "sourceVolume": "vol00",
          "readOnly": null
        }
      ],
      "name": "elasticsearch",
      "ulimits": null,
      "dockerSecurityOptions": null,
      "environment": [
        {
          "name": "ES_HEAP_SIZE",
          "value": "1g"
        }
      ],
      "links": null,
      "workingDirectory": null,
      "readonlyRootFilesystem": null,
      "image": "daptiv/elasticsearch-ecs:latest",
      "command": [
        "/docker-entrypoint.sh",
        "--discovery.type=ec2",
        "--discovery.ec2.groups={{ name or id of security group assigned to ec2 instances in the cluster ie. sg-1a2b3c4d }}"
      ],
      "user": null,
      "dockerLabels": null,
      "logConfiguration": null,
      "cpu": 300,
      "privileged": null,
      "expanded": true
    }
  ],
  "volumes": [
    {
      "host": {
        "sourcePath": "/var/data/vol00"
      },
      "name": "vol00"
    }
  ],
  "family": "elasticsearch"
}
```


### Critical Parts of the Sample Task Definition

##### Mount points

If an ebs volume is to be used for elasticsearch data storage (by default it will use the docker storage on the ECS AMI, mounted at /dev/xvdcz), it should be configured here:  

```
"mountPoints": [
  {
    "containerPath": "/usr/share/elasticsearch/data",
    "sourceVolume": "vol00",
    "readOnly": null
  }
]
```

and/or if you wish to provide a configuration file (elasticsearch-config references a elasticsearch config file on the EC2 host, which was copied from s3 in the EC2 intance user data):

```      
"mountPoints": [
  {
    "containerPath": "/usr/share/elasticsearch/config/elasticsearch.yml",
    "sourceVolume": "elasticsearch-config",
    "readOnly": null
  }
]
```


##### Command

these settings can be required as a command or in the elasticsearch.yml config file.  However they are required for containers destributed accross EC2 instances to form a cluster.  This is part of the aws-cloud plugin and is documented here: https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/cloud-aws-discovery.html

```
"command": [
  "/docker-entrypoint.sh",
  "--discovery.type=ec2",
  "--discovery.ec2.groups={{ name or id of security group assigned to ec2 instances in the cluster ie. sg-1a2b3c4d }}"
]
```


##### Port Mapping

elasticsearch uses port 9300 for communication within the cluster, and exposes it's api on port 9200 - so these will need to be mapped to the container host.

```
"portMappings": [
  {
    "hostPort": 9200,
    "containerPort": 9200,
    "protocol": "tcp"
  },
  {
    "hostPort": 9300,
    "containerPort": 9300,
    "protocol": "tcp"
  }
]
```


##### Environment

the installations heap size (1g by default) can be configured via the ES_HEAP_SIZE environment variable for the container.  The reserved memory of the container must be larger than the ES_HEAP_SIZE.  Recommendations about what to set this value as can be found here: https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html
 
```
"environment": [
  {
    "name": "ES_HEAP_SIZE",
    "value": "1g"
  }
]
```


### Optional Configuration

The following can be configured in the elasticsearch.yml configuration file if required.


##### Node Type

If the cluster grows to be larger than a couple of nodes, nodes should be designated as either designated master or dedicated data nodes for cluster stability.  See: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html

```
# Allow this node to be eligible as a master node (enabled by default):
node.master: true
# Allow this node to store data (enabled by default):
node.data: true
```


##### Cluster Name

If multiple clusters are running on the same security group, a name can be specified for each cluster so that they stay seperated during aws discovery.

```
cluster.name: gra-elk
```


##### Index Configuration

The number of replicas and shards per index should be altered away from the default for larger clusters.  Search performance and availablity is increased with more replication, while an increase in sharding will enhance indexing performance and are a factor in if and how an index can be split accross nodes in the cluster.

```
# Set the number of shards (splits) of an index (5 by default):
index.number_of_shards: 5
# Set the number of replicas (additional copies) of an index (1 by default):
index.number_of_replicas: 1
```


##### Network Host

This is present in the official elasticsearch container's elasticsearch.yml, and seems to be required for ec2 discovery to function correctly.

```
# This is in the default elasticsearch.yml and seems to be needed for aws descovery
network.host: 0.0.0.0
```


##### Minimum Master Nodes

When running more than 2 nodes in a cluster, the minimum number of master nodes must be set in order to mitigate the risk of running into the split brain problem.  See: https://www.elastic.co/guide/en/elasticsearch/guide/1.x/_important_configuration_changes.html#_minimum_master_nodes

```
discovery.zen.minimum_master_nodes: 2
```


##### Memory

Swapping memory to disk can cause performance issues in a cluster. the memory can be locked using the bootstrap.mlockall option in the configuration file.  See: https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html#_swapping_is_the_death_of_performance

```
bootstrap.mlockall: true
```  

##### Name

If better names are needed than the default superhero names, they can be set using the node.name property.

```
node.name: "i-12345678"
```

Or, to set it as the ID of the EC2 instance hosting the container, this can be done in the ec2 user data.

```
idDocument=http://169.254.169.254/latest/dynamic/instance-identity/document  
RESOURCE_ID=`curl -s $idDocument | grep instanceId | cut -d \" -f4`  
echo $'\n'node.name: \"$RESOURCE_ID\" >> {{ Elasticsearch config yml file location }}
```
