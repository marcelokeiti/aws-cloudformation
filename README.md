# aws-cloudformation
Here you will find a suggested architecture to deploy all your resources in AWS.

## Infrastructure
This template aimed to create the underline infrastructure in which other layers will be deployed.

![Infrastructure](/infrastructure/resources/infrastructure.jpg) 

## WebApp Enviroment
This template aimed to create the components related to a web application.

![WebApp Enviroment](/webapp_env/resources/webapp-env.jpg) 

#### Elastic Load Balancing
This snippet creates an Elastic Load Balancing allowed to redirect to all 3 public subnets created by the Infrastructure Cloud Formation template.
```
  ELB:
    Type: "AWS::ElasticLoadBalancing::LoadBalancer"
    Properties:
      CrossZone: true
      Listeners:
        -
          InstancePort: "80"
          InstanceProtocol: HTTP
          LoadBalancerPort: "80"
          Protocol: HTTP
      Subnets:
        - !ImportValue infra-public-subnet-a
        - !ImportValue infra-public-subnet-b
        - !ImportValue infra-public-subnet-c
      SecurityGroups:
        - !Ref ELBSecurityGroup
```

#### Launch Configuration
This snippet creates a Launch Configuration which is configuration used to launch new instances controlled by the Auto Scaling mechanism. In this case we are configuring to launch an image passed by parameter (AMI, InstanceType, KeyName). 

`UserData` Executed after launching the instance:  
`/opt/aws/bin/cfn-init` It executes the `Metadata` instructions  
`/opt/aws/bin/cfn-signal` If previous command was success It signal AutoScalingGroup, otherwise the AutoScalingGroup will not receive this signal and will eventually rollback  

```
  LaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Properties:
      ImageId: !Ref AMI
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SecurityGroups:
        - !Ref LCSecurityGroup
      UserData:
        "Fn::Base64":
          !Sub |
            #!/bin/bash
            yum update -y aws-cfn-bootstrap # good practice
            /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource LaunchConfiguration --configsets init_all --region ${AWS::Region}
            yum -y update
            /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          init_all:
            - "configure_cfn"
            - "install_www"
        configure_cfn:
          files:
            /etc/cfn/hooks.d/cfn-auto-reloader.conf:
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.LaunchConfiguration.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource LaunchConfiguration --configsets init_all --region ${AWS::Region}
              mode: "000400"
              owner: root
              group: root
            /etc/cfn/cfn-hup.conf:
              content: !Sub |
                [main]
                stack=${AWS::StackId}
                region=${AWS::Region}
                verbose=true
                interval=5
              mode: "000400"
              owner: root
              group: root
          services:
            sysvinit:
              cfn-hup:
                enabled: "true"
                ensureRunning: "true"
                files:
                  - "/etc/cfn/cfn-hup.conf"
                  - "/etc/cfn/hooks.d/cfn-auto-reloader.conf"
        install_www:
          packages:
            yum:
              httpd: []
          services:
            sysvinit:
              httpd:
                enabled: "true"
                ensureRunning: "true"
```

`MetaData` Instructions to install and configure packages, files, users  
`cfn-hup` Pooling to identify changes in the Cloud Formation template  
`cfn-auto-reloader.conf` Re-execute `cfn-init` whenever `cfn-hup` identifies a change in the Cloud Formation template  
`install_www` Install apache httpd  


#### Auto Scaling Group
This snippet creates an Auto Scaling Group which controls the scale-in, scale-out of the cluster. It uses the Launch Configuration to instanciate instances.

`DesiredCapacity` If the amount of signal received is less than the desired capacity It will rollback.
`UpdatePolicy` The Launch Configuration cannot be changed. To update It, the AWS needs to replace It with a new Launch Configuration. As a result the Auto Scaling Group is updated since It needs to reference the new Launch Configuration. This policy tells the Cloud Formation if the existent instances needs to be replaced every time an update occurs.

```
  AutoScalingGroup:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    CreationPolicy:
      ResourceSignal:
        Count: !Ref DesiredCapacity
        Timeout: "PT5M"
    UpdatePolicy:
      AutoScalingReplacingUpdate:
        WillReplace: true
    Properties:
      LaunchConfigurationName: !Ref LaunchConfiguration
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      Cooldown: "300"
      HealthCheckGracePeriod: "300"
      HealthCheckType: ELB
      LoadBalancerNames:
        - !Ref ELB
      VPCZoneIdentifier:
        - !ImportValue infra-public-subnet-a
        - !ImportValue infra-public-subnet-b
        - !ImportValue infra-public-subnet-c
```




