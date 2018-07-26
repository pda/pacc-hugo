---
title: "MailHog on ECS + Service Discovery + CloudFormation"
date: 2018-07-24T23:15:27+10:00
---

AWS [announced Service Discovery for ECS](https://aws.amazon.com/blogs/aws/amazon-ecs-service-discovery/) a few months ago. It builds upon the [Route 53 Auto Naming API announced](https://aws.amazon.com/about-aws/whats-new/2017/12/amazon-route-53-releases-auto-naming-api-name-service-management/) a few months before that.

Clear documentation and examples for using it with CloudFormation are currently hard to find.  Here's a small but complete example of an ECS Service on Fargate using Service Discovery for internal and external discovery.

I'm using the fake/development mail server [MailHog](https://github.com/mailhog/MailHog) as an example. Thanks to David&nbsp;Looi at Culture&nbsp;Amp for his contributions to this.

This stack creates;

* A Fargate cluster,
* MailHog ECS service running four containers,
* MongoDB ECS service running one container,
* Service Discovery Private DNS Namespace; `mail.acme`,
* Service Discovery Service for MailHog; `smtp.mail.acme`,
* Service Discovery Service for MongoDB; `mongo.mail.acme`.

The MailHog service will use `mongo.mail.acme` service discovery to locate MongoDB. And our external caller will use `smtp.mail.acme` to locate (and DNS-load-balance between) the MailHog services.  We'll test it via SMTP and HTTP API from an EC2 instance in the same VPC.

Here's what we're aiming for:

![](/ecs-service-discovery-cloudformation/architecture-diagram.png)

And here's the CloudFormation template:

Pay special attention to the `HealthCheckCustomConfig` property on `AWS::ServiceDiscovery::Service` — it's non-obvious, but omitting it leads to mysterious stack timeouts as describe in [this AWS forum thread](https://forums.aws.amazon.com/thread.jspa?threadID=283572).

```yaml
Description: MailHog on ECS with Service Discovery

Parameters:

  SourceSecurityGroup:
    Type: AWS::EC2::SecurityGroup::Id

  Subnets:
    Type: List<AWS::EC2::Subnet::Id>

  VPC:
    Type: AWS::EC2::VPC::Id

Resources:

  Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Ref AWS::StackName

  MailService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !Ref Cluster
      LaunchType: FARGATE
      DesiredCount: 4
      ServiceName: mailhog-mail
      ServiceRegistries:
        - RegistryArn: !GetAtt MailServiceDiscovery.Arn
      TaskDefinition: !Ref MailTaskDefinition
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref Subnets
          SecurityGroups:
            - !GetAtt MailServiceSecurityGroup.GroupId

  MailServiceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: mailhog ECS task
      SecurityGroupIngress:
        - {ToPort: 1025, FromPort: 1025, IpProtocol: tcp, SourceSecurityGroupId: !Ref SourceSecurityGroup} # HTTP
        - {ToPort: 8025, FromPort: 8025, IpProtocol: tcp, SourceSecurityGroupId: !Ref SourceSecurityGroup} # SMTP
      VpcId: !Ref VPC

  MailTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      RequiresCompatibilities: ["FARGATE"]
      NetworkMode: awsvpc
      Cpu: "256"
      Memory: "0.5GB"
      Family: !Ref AWS::StackName
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      ContainerDefinitions:
        - Name: mailhog
          Image: mailhog/mailhog:latest
          Environment:
            - {Name: MH_STORAGE, Value: "mongodb"}
            - {Name: MH_MONGO_URI, Value: "mongo.mail.acme:27017"}

  MongoService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !Ref Cluster
      LaunchType: FARGATE
      DesiredCount: 1
      ServiceName: mailhog-mongo
      ServiceRegistries:
        - RegistryArn: !GetAtt MongoServiceDiscovery.Arn
      TaskDefinition: !Ref MongoTaskDefinition
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref Subnets
          SecurityGroups:
            - !GetAtt MongoServiceSecurityGroup.GroupId

  MongoServiceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: mailhog ECS task
      SecurityGroupIngress:
        - {ToPort: 27017, FromPort: 27017, IpProtocol: tcp, SourceSecurityGroupId: !Ref MailServiceSecurityGroup}
      VpcId: !Ref VPC

  MongoTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      RequiresCompatibilities: ["FARGATE"]
      NetworkMode: awsvpc
      Cpu: "256"
      Memory: "0.5GB"
      Family: !Sub ${AWS::StackName}-mongo
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      ContainerDefinitions:
        - Name: mongo
          Image: mongo:latest

  TaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - {Action: "sts:AssumeRole", Effect: Allow, Principal: {Service: ecs-tasks.amazonaws.com}}
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

  ServiceDiscoveryNamespace:
    Type: AWS::ServiceDiscovery::PrivateDnsNamespace
    Properties:
      Name: mail.acme
      Vpc: !Ref VPC

  MailServiceDiscovery:
    Type: AWS::ServiceDiscovery::Service
    Properties:
      Name: smtp
      DnsConfig:
        DnsRecords: [{Type: A, TTL: "10"}]
        NamespaceId: !Ref ServiceDiscoveryNamespace
      HealthCheckCustomConfig:
        FailureThreshold: 1

  MongoServiceDiscovery:
    Type: AWS::ServiceDiscovery::Service
    Properties:
      Name: mongo
      DnsConfig:
        DnsRecords: [{Type: A, TTL: "10"}]
        NamespaceId: !Ref ServiceDiscoveryNamespace
      HealthCheckCustomConfig:
        FailureThreshold: 1
```

Fill in the blanks and launch the stack:

```sh
aws cloudformation create-stack \
  --stack-name pda-mailhog \
  --capabilities CAPABILITY_IAM \
  --template-body="file://cloudformation.yaml" \
  --parameters \
    "ParameterKey=SourceSecurityGroup,ParameterValue=____" \
    "ParameterKey=Subnets,ParameterValue=____" \
    "ParameterKey=VPC,ParameterValue=____"
```

## Demo from EC2 in same VPC

Check the DNS provided by Service Discovery. There's four ECS Tasks, so four `A` records.

```plain
[ec2-user@... ~]$ dig a smtp.mail.acme
...
;; QUESTION SECTION:
;smtp.mail.acme.                        IN      A

;; ANSWER SECTION:
smtp.mail.acme.         10      IN      A       10.0.147.126
smtp.mail.acme.         10      IN      A       10.0.150.59
smtp.mail.acme.         10      IN      A       10.0.145.36
smtp.mail.acme.         10      IN      A       10.0.147.90
...
```

Check MailHog's API. The DNS resolver will pick one random-ish-ly:

```plain
[ec2-user@... ~]$ curl -s http://smtp.mail.acme:8025/api/v2/messages | jq .
{
  "total": 0,
  "count": 0,
  "start": 0,
  "items": []
}
```

Deliver a message:

```plain
[ec2-user@... ~]$ nc -C smtp.mail.acme 1025
220 mailhog.example ESMTP MailHog
HELO pda
250 Hello pda
MAIL FROM:<paul@example.com>
250 Sender paul@example.com ok
RCPT TO:<melbourne@awsug.org.au>
250 Recipient melbourne@awsug.org.au ok
DATA
354 End data with <CR><LF>.<CR><LF>
From: "Paul Annesley" <paul@example.com>
To: <melbourne@awsug.org.au>
Date: Tue, 24 July 2018 23:10:00 +1000
Subject: Culture Amp is hiring!

Hello world!
.
250 Ok: queued as 2tNO5vtw8iium72EOzEP16TzYLg1ybYucF6E9wWOU1k=@mailhog.example
QUIT
221 Bye
```

Check MailHog's API again:

```plain
[ec2-user@... ~]$ curl -s http://smtp.mail.acme:8025/api/v2/messages \
  | jq '.items[0].Content.Headers.Subject[0]'
"Culture Amp is hiring!"
```
