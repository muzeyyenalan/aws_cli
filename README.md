###### LAUNCH INSTANCE WİTH CLI COMMAND

You will use the **AWS Systems Manager Parameter Store** to obtain the ID of the most recent *Amazon Linux 2* AMI. AWS maintains a list of standard AMIs in the Parameter Store, making this task easy to automate.

```
# Set the Region
AZ=`curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone`
export AWS_DEFAULT_REGION=${AZ::-1}
# Obtain latest Linux AMI
AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --query 'Parameters[0].[Value]' --output text)
echo $AMI
```

This command did the following:

* Obtained the Region where the instance is running
  * Called the AWS Systems Manager ( *ssm* ) and used the **get-parameters** command to retrieve a value from the Parameter Store
* The AMI requested was for Amazon Linux 2 ( *amzn2-ami* )The AMI ID has been stored in an Environment Variable called *AMI*
  If your SSH session disconnects, it will lose the information stored in environment variables. Once you reconnect, you will need to re-run all of the steps in this task, starting with the above commands to obtain the AMI ID.

### Obtain the Subnet to Use

You will be launching the new instance in the Public Subnet. When launching an instance, the **SubnetId** can be specified.

The following command will retrieve the *SubnetId* for the Public Subnet:

```
SUBNET=$(aws ec2 describe-subnets --filters 'Name=tag:Name,Values=Public Subnet' --query Subnets[].SubnetId --output text)
echo $SUBNET
```

This uses the AWS CLI to retrieve the Subnet ID of the subnet named  *Public Subnet* .

### Obtain the Security Group to Use

A *Web Security Group* has been provided which allows inbound HTTP requests.

```
SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=WebSecurityGroup --query SecurityGroups[].GroupId --output text)
echo $SG
```

The command retrieves the *Security Group ID* of the Web Security Group.


### Launch the Instance

```
INSTANCE=$(\
aws ec2 run-instances \
--image-id $AMI \
--subnet-id $SUBNET \
--security-group-ids $SG \
--user-data file:///home/ec2-user/UserData.txt \   #UserData created in UserData.txt
--instance-type t3.micro \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Web Server}]' \
--query 'Instances[*].InstanceId' \
--output text \
)
echo $INSTANCE
```


### Wait for the Instance to be Ready


All information related to the instance will be displayed in JSON format. Amongst this information is the instance status.


```
aws ec2 describe-instances --instance-ids $INSTANCE
```

Specific information can be obtained by using the **query** parameter.

```
aws ec2 describe-instances --instance-ids $INSTANCE --query 'Reservations[].Instances[].State.Name' --output text
```


### Test the Web Server

You can now test that the web server is working. You can retrieve a URL to the instance via the AWS CLI.

```
aws ec2 describe-instances --instance-ids $INSTANCE --query Reservations[].Instances[].PublicDnsName --output text
```

This returns the **DNS Name** of the instance.
