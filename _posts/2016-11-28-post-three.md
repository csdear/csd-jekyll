---
layout: post
title: "SSL Certificate Creation via AWS"  
published: true
category: AWS		
author: "csdear"
---



<img src="/images/redacted/wall-of-keys.jpg" alt="keys" style="width:580px;height:300px;margin-left:40px;" />
<br />
Here's some things I learned and a general roadmap for setting up and configuring a SSL certificate through AWS via a load balancer.
You implementation needs may vary, so this documention should serve as a general path toward securing a site with AWS SSL generated certificates.  

AWS's Certificate Manager (ACM) service can be used to secure sites with an SSL certificate.  
The SSL certificates are free --  though there is some underlying costs with using a load balancer. See 
[Link](https://aws.amazon.com/elasticloadbalancing/classicloadbalancer/pricing/) for more information.


* ACM certificates are domain validated, e.g., example.com
* Wildcards can be used to to protect sever sites under the same domain, e.g.  .example.com would protect www.example.com as well as  images.example.com etc.
* Ownership of the domain will need to be validated.  Email will be sent to the registered owner for each domain name in the request.
* SSL Validity period is currently 13 months.  However, AWS makes it easy with Managed Renewal means less downtime due to misconfigured, revoked or expired certificates... SSL Certs can be pain to keep up with and configure correctly.
* ACM attempts to auto renew certificates before they expire. If it can't, it should send a validation email.
* ACM is INTEGRATED with AWS services , so you can centrally manage and deploy ACM certificates for your web applications hosted on the AWS platform.


<br />
**AWS SSL Certificates can only be deployed on a Load Balancer or Cloudfront instance.**  
	 You do not install your ACM Certificate directly on the Amazon EC2 instances that contain your website or your application. Instead, you associate the ACM Certificate with an AWS service, such as Elastic Load Balancing or Cloudfront instance.  
	 For this documentation, we will be putting a SSL cert on a a ELB instance we created.  Currently, we are creating ELBs for each site we secure a SSL cert for, as the SSL is registered to a singular domain.

**Cloudfront Implementation** : The second option to setup a SSL Certificate via Cloudfront is not covered here. However, this is a good solution if you have a simple site hosted within a S3 bucket and is easier to configure that the Load Balancer route.    Example tutorial :  https://deliciousbrains.com/wp-offload-s3/doc/custom-domain-https-cloudfront/

**STEPS OVERVIEW**

* Setup SSL certificate with AWS Certificate Manager
* Create a ELB Load Balancer to deploy the SSL Certificate to
* Configure Route 53 to route to the load balancer
* Test and verify SSL Certificate implementation
<br />

*See also handy reference* [Creating and Elastic Load Balancer](http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-create-https-ssl-load-balancer.html#create-https-lb-console)
<br />
<br />

##SETTING UP THE CERTIFICATE with AMAZON CERTIFICATE MANAGER

1. Go to AWS Certificate Manager ACM and click on Certificate Manager
2. Get Started
3. Add Domain Names
	* For example, *.example.com, which would secure api.example.com as well.  Note that sub domains can be added and protected with the same SSL cert by using wildcards.  e.g *.mysite.com would cover qa.mysite.com or uat.mysite.com as well
4. Select 'Review and Request'
	* Email is sent to the registered owner of each domain.  No worries if you do not know the email address tied to the domain -- this information will be displayed to you on the verification page summary later. Else you could always perform a WHOIS lookup. 
	![Review](/images/redacted/p1.png)
5. Verification is deployed to the email(s) attached to the the domain. The email addresses the verificaiton is sent to is displayed :   
	![validation](/images/redacted/p2.png)
	
6. Await verification from account owner before moving to next step.  After it has been verified, status will change to 'Issued' 
![Issued](/images/redacted/p3.png)

##DEPLOYING THE SSL CERTIFICATION

6. Setup a load balancer for the site.  Go to EC2>> and click the 'Load Balancer' link
7. Select Button ' Create Load Balancer'
8. Select create 'Classic Load Balancer' 
9. Create a Name for the LB : e.g. api-example-elb
10. Select the private cloud the LB is hosted inside : e.g.,  vpc-3j67965z (42.0.0.0/16)
11. Select Scheme : Internet-Facing
12. Listeners : Add the listeners
	*  HTTP : 80
	*  HTTPS : 443

13. Select the Subnets of Availability Zones. These exist within your VPC. 
  
14. Select the available subnets
	a. e.g., us-east-1a | subnet-5fezuz19
	
15. Next Select Security Groups : 
	*  For example, my LB uses two security groups.  Inbound rules for the security group to accept connections via HTTP (80) (to connect to the load balancer) and HTTPS (443) need to be setup.   
		*  sg-12345678, prod-web-sg.  Production web security group.
		*  sg-12345610, prod-elb-sg.  Production elb.  

		Example inbound and outbound rules for a security group:
		![sg rules](/images/redacted/p45.jpg)

16. Select the SSL Certificate to use.  Here we select the SSL certificate we configured in ACM: 
![select](/images/redacted/p6.png)

17. Select the security policy 
	a. Select the predfined security policy'ELBSecurityPolicy-2016-08 
	b.  Next you are asked to Create or Select a security group.   
18. Add EC2 Instances
	*  Select the EC2 instance wherein your site resides
	*  Check 'Enable Cross Zone Load Balancing' 
	*  Check 'Enable Connection Draining, 300 seconds'  

19. Click Next, to step 6 : Review.  Note all the load balancer settings for later.  

20. Select 'Create': 
![Create](/images/redacted/p7.png)  

21. Load balancer created.  Status will be 'provisioning' and eventually change to 'active'.  

22. On Load Balancer view, copy down the 'DNS name'.  You might need it later when configuring Route 53:<br /> ***DNS name : api-example-elb-123456789.us-east-1.elb.amazonaws.com (A Record)***

23. Port Configuration 
	*  On the Description tab of you load balancer the following ports should be configured :
	![Port Configuration](/images/redacted/p8.png)

24. Instances Tab
	*  On the instances tab, the EC2 instance wherein in you target site resides should be listed.  
	*  Verify Availability zone you had selected prior is configured, e.g., 'us-east-1a'/subnet-5fezuz19 is configured.

24. Healthcheck Tab :  
	*  We currently have the following configured for a health check which checks the default document on the server : 
	![Health Check](/images/redacted/p9.png)

24. **A Note on 'Target Groups'**
NOTE : this feature was not needed for this implementation but might be needed for yours.  
It turned out we did not need any EC2 targets registered in order to get a SSL cert going, but be advised your implementation might need to configure them.  
See [Target Groups for Your Application Load Balancers](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#registered-targets) for more information regarding registereding EC2 Targets.

##CONFIGURE ROUTE 53 
27. Next , go to Route 53
28. Select DNS Management : Hosted Zones
29. Select the hosted zone your site is configured in.  Lets say you have a hosted zone for "example.com"
30. Select Create Record Set
31. Enter the name of the record set : e.g., api.example.com
32. Type, A -- IPv4 address
	a. Note, the AWS documentation usually advises to create a CNAME with the elb dns url pasted in.  However for this implementation we used a Type - A record with a Alias Target and it worked as expected.  
33. Select the Alias : Yes radio button
34. Set the Alias Target
	Either paste in the ELB DNS entry you noted earlier, or select the elb from the dropdown when clicking the Alias Target field:
	![Alias Target](/images/redacted/p10.png)  

***NOTE :*** Another option is to use a CNAME instead.  This implementation uses a A-IPv4 address with an alias target to the ELB.  However, there are many examples out there that use a CNAME type instead.  With that implementation, it is possible to route a DNS name (example.com) to another 'target' DNS name (elb12344.elb.amazoneaws.com).  

##TESTING THE DEPLOYMENT
Internal / External to your work network : Be aware when testing that you can get different results testing internally on your internal network at work verses externally.  It is best to verify the implementation of a SSL certificate outside of our network.

I. Testing Via iphone app 'Cert Inspector'
![Create](/images/redacted/p11.png)  

II. Testing via online tool SSL Checker : [ssl checker](https://www.sslshopper.com/ssl-checker.html)

III.  Testing via Terminal DIG command (Mac/Linux Terminal)
![Create](/images/redacted/p12.png)   


Yes but how do you know that is the ELB?  
Because the IP addresses match if you dig the elb -- dig @8.8.8.8 api-example-elb-555315498.us-east-1.elb.amazonaws.com :
![Create](/images/redacted/p13.png)  

That's it.  I hope you find my example path helpful in setting up your own SSL certificates via AWS.


 
 