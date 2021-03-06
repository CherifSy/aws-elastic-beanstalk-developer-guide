# Configuring End\-to\-End Encryption in a Load Balanced Elastic Beanstalk Environment<a name="configuring-https-endtoend"></a>

Terminating secure connections at the load balancer and using HTTP on the backend may be sufficient for your application\. Network traffic between AWS resources cannot be listened to by instances that are not part of the connection, even if they are running under the same account\.

However, if you are developing an application that needs to comply with strict external regulations, you may be required to secure all network connections\. You can use configuration files to make your Elastic Beanstalk environment's load balancer connect to backend instances securely to meet these requirements\.

First add a secure listener to your load balancer, if you haven't already:

**`.ebextensions/https-lbterminate.config`**

```
option_settings:
  aws:elb:listener:443:
    SSLCertificateId: arn:aws:acm:us-east-2:#############:certificate/####################################
    ListenerProtocol: HTTPS
```

You must also configure the instances in your environment to listen on the secure port and terminate HTTPS connections\. The configuration varies per platform\. See  for instructions\. You can use a self\-signed certificate for the EC2 instances without issue\.

Next, configure the listener to forward traffic using HTTPS on the secure port used by your application\. You can also change the default health check to use this port and protocol to ensure that the load balancer is able to connect securely\. The following configuration file does both:

**`.ebextensions/https-reencrypt.config`**

```
option_settings:
  aws:elb:listener:443:
    InstancePort: 443
    InstanceProtocol: HTTPS
  aws:elasticbeanstalk:application:
    Application Healthcheck URL: HTTPS:443/
```

**Note**  
The EB CLI and Elastic Beanstalk console apply recommended values for the preceding options\. You must remove these settings if you want to use configuration files to configure the same\. See  for details\.

The next part is a bit more complex\. You need to modify the load balancer's security group to allow traffic, but depending on whether you launch your environment in the default VPC or a custom VPC, the load balancer's security group will vary\. In a default VPC, Elastic Load Balancing provides a default security group that can be used by all load balancers\. In a VPC that you create, Elastic Beanstalk creates a security group for the load balancer to use\.

To support both scenarios, you can create a security group and tell Elastic Beanstalk to use that\. The following configuration file creates a security group and attaches it to the load balancer:

**`.ebextensions/https-lbsecuritygroup.config`**

```
option_settings:
  # Use the custom security group for the load balancer
  aws:elb:loadbalancer:
    SecurityGroups: '`{ "Ref" : "loadbalancersg" }`'
    ManagedSecurityGroup: '`{ "Ref" : "loadbalancersg" }`'

Resources:
  loadbalancersg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: load balancer security group
      VpcId: vpc-########
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

Replace the highlighted text with your default or custom VPC ID\. The above example includes ingress and egress over port 80 to allow HTTP connections\. You can remove those properties if you only want to allow secure connections\.

Finally, add ingress and egress rules that allow communication over 443 between the load balancer's security group and the instances' security group:

**`.ebextensions/https-backendsecurity.config`**

```
Resources:
  # Add 443-inbound to instance security group (AWSEBSecurityGroup)
  httpsFromLoadBalancerSG: 
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: {"Fn::GetAtt" : ["AWSEBSecurityGroup", "GroupId"]}
      IpProtocol: tcp
      ToPort: 443
      FromPort: 443
      SourceSecurityGroupId: {"Fn::GetAtt" : ["loadbalancersg", "GroupId"]}
  # Add 443-outbound to load balancer security group (loadbalancersg)
  httpsToBackendInstances: 
    Type: AWS::EC2::SecurityGroupEgress
    Properties:
      GroupId: {"Fn::GetAtt" : ["loadbalancersg", "GroupId"]}
      IpProtocol: tcp
      ToPort: 443
      FromPort: 443
      DestinationSecurityGroupId: {"Fn::GetAtt" : ["AWSEBSecurityGroup", "GroupId"]}
```

Doing this separately from security group creation allows you to restrict the source and destination security groups without creating a circular dependency\.

With all of the above pieces in place, the load balancer connects to your backend instances securely Using HTTPS\. The load balancer doesn't care if your instance's certificate is self\-signed or issued by a trusted certificate authority, and will accept any certificate presented to it\.

You can change this by adding policies to the load balancer that tell it only to trust a specific certificate\. The following configuration file creates two policies\. One policy specifies a public certificate, and the other tells the load balancer to only trust that certificate for connections to instance port 443:

**`.ebextensions/https-backendauth.config`**

```
option_settings:
  # Backend Encryption Policy
  aws:elb:policies:backendencryption:
    PublicKeyPolicyNames: backendkey
    InstancePorts:  443
  # Public Key Policy
  aws:elb:policies:backendkey:
    PublicKey: |
      -----BEGIN CERTIFICATE-----
      ################################################################
      ################################################################
      ################################################################
      ################################################################
      ################################################
      -----END CERTIFICATE-----
```

Replace the highlighted text with the contents of your EC2 instance's public certificate\.