# Jenkins BlueOcean on ECS
Instruction and a configuration file to setup Jenkins server on AWS ECS.

## Summary
* Uses [SpyceTek's Jenkins Docker container](https://cloud.docker.com/u/spycetek/repository/docker/spycetek/jenkins-blueocean) based on Jenkins BlueOcean container.
  It is installed AWS CLI tools to the base image.
* Runs the container from AWS ECS.
* Makes the Jenkins configurations and output files parmanent with EFS.
  Jenkins stores all data as files, not database.
* Uses EC2 type ECS, since Fargate does not support EFS. (as of 2019/07/29)  
  (Fargate automatically creates EC2 instances, CloudWatch Logs, and public IPs for you, but no luck.)
* Uses default cluster on default VPC.
* Assigns elastic IP to the EC2 instance to keep the same IP.
* Uses CloudWatch logs (awslogs driver).


## Setup Instruction
1. Create EFS to mount to Docker container.
2. Start EC2 instance that runs Jenkins container.
3. Register & start ECS task (Jenkins container).


### Create EFS
Just create by following the wizard from the AWS EFS console. All default is fine.

After creationg, change `device` and `o` in jenkins-task.json file with the
domain name of the created EFS.

### Create CloudWatch Logs
Create log group with the name specified in jenkins-task.json file.  
Change `awslogs-region` in the json file and the command below, according to your region.

```
aws logs create-log-group --log-group-name "/ecs/jenkins" --region ap-southeast-1
```

If you don't want to use it, skip this and aremove the related lines from jenkins-task.json file.

[CloudWatch Reference](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html)


### Prepare an EC2 Instance
[Setting Up with Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/get-set-up-for-amazon-ecs.html)

* Create a roll "ecsInstanceRole".
  Refer to ["To create the ecsInstanceRole IAM role for your container instances"](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html)
* Start EC2 instance with amazon-ecs-optimized AMI from the link in the table of [this page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI_EC2.html).
  * t2.micro seems to be enough, and `cpu` and `memoryReservation` in jenkins-task.json are
    optimized for t2.micro. If you change the instance type, you should adjust `cpu` and
    `memoryReservation` as well.
  * Disable "Auto-assign Public IP", because we want to use elastic IP
    to avoid the IP is changed every time the instance is changed.
  * Use `ecsInstanceRole` created above.
  * Remove storage except the root (8Gib), because we will mount EFS.
  * Create security groups to allow inbount access to the ports bellow;
    * host 80 to container 8080 (Jenkins)
    * host 50000 to container 50000 (Jenkins)
    * host 22 to container 22 (SSH)
  * Add security group that is allowed to access test database of test web server.
* Create elastic IP and assign it to this new EC2 instance.
  Note: after IP is assigned, the instance will be registered in the default cluster.
  You can see it in in ECS console page.


### Start Jenkins Container on ECS
#### Create Role
Create `ecsTaskExecutionRole` by following "To create the ecsTaskExecutionRole IAM role" section in ["Amazon ECS Task Execution IAM Role" page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html).

Change `executionRoleArn` in jenkins-task.json file to the arn of the role you just created.

#### Register the Task
```
aws ecs register-task-definition --cli-input-json file://./jenkins-task.json
```

Check if it is successfully registered.

```
aws ecs list-task-definitions
```

#### Run the Task
```
aws ecs run-task --cluster default --task-definition jenkins:1 --count 1
```

Specify the task by `--task-definition`. Make sure the version is the latest.
The version is the number after the task name as in `jenkins:1`.

**Note**: If the instance type is t2.micro, it takes 15 minutes to boot up at the first time.

Check if the task is running.

```
aws ecs list-tasks
```

To stop, execute the command below specifying your task arn.

```
aws ecs stop-task --cluster default --task arn:aws:ecs:ap-southeast-1:078106402841:task/dcc6b66c-3f9d-43be-a9fe-dbf25921ab1b
```


#### Notes Creating Task Definitions
* [AWS ECS Reference](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/Welcome.html)
* Attaching EFS as volumen
  * https://stackoverflow.com/a/52671766/1646699  
    `device` part seems wrong. So, also take look at the page below;
  * https://garbe.io/blog/2018/09/12/the-easiest-way-to-use-efs-volumes-with-ecs/
  * https://docs.aws.amazon.com/efs/latest/ug/mounting-fs.html
* It is also possible to mount EFS to EC2 instance, and configure the volume in normal way.  
  * https://qiita.com/tandfy/items/829f9fcc68c4caabc660
  * https://aws.amazon.com/blogs/compute/using-amazon-efs-to-persist-data-from-amazon-ecs-containers/

## Applying TSL to Jenkins Server
**TODO**: Apply TSL to Jenkins server.

It seems easiest way is to use the reverse proxy of Nginx.  
https://itnext.io/setting-up-https-for-jenkins-with-nginx-everything-in-docker-4a118dc29127
