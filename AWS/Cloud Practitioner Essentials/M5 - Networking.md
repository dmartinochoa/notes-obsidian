### **Recap and next steps**

In this networking module, you identified core networking components and how they connect in the AWS Cloud. We covered the basics of a VPC, the way that you isolate your workload in AWS, gateways, network ACLs, and security groups. You also reviewed ways to connect to AWS through a VPN and Direct Connect, secure connections that are either encrypted over the public internet or exclusive connections used by you and you alone.

You also learned about AWS edge locations, Route 53 for DNS, and CloudFront to cache content closer to consumers.

### Resources

To learn more about the material covered in this module, choose the resource links in the following table.

|Resource link|Description|
|---|---|
|[Amazon Virtual Private Cloud(opens in a new tab)](https://aws.amazon.com/vpc/)|Amazon VPC is a service to provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define.|
|[Subnet(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)|A subnet is a section of a VPC that can contain resources and is used to organize your resources. They can contain be either public or private.|
|[Internet gateway(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)|An internet gateway is a connection between a VPC and the internet. It allows public traffic from the internet to access your VPC.|
|[Virtual private gateway(opens in a new tab)](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html#VPNGateway)|A virtual private gateway is the component that allows protected internet traffic to enter into the VPC. It allows a connection between your VPC and a private network only if it is coming from an approved network.|
|[AWS Client VPN(opens in a new tab)](https://aws.amazon.com/vpn/client-vpn/)|Amazon Client VPC is a networking service you can use to connect your remote workers and on-premises networks to the cloud. It is a fully managed, elastic VPN service that automatically scales up or down based on user demand.|
|[AWS Site-to-Site VPN(opens in a new tab)](https://aws.amazon.com/vpn/site-to-site-vpn/)|AWS Site-to-Site VPN creates a secure connection between your data center or branch offices and your AWS Cloud resources.|
|[AWS PrivateLink(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)|AWS PrivateLink is a highly available, scalable technology that you can use to privately connect your VPC to services and resources as though they were in your VPC.|
|[AWS Direct Connect(opens in a new tab)](https://aws.amazon.com/directconnect/)|AWS Direct Connect is a service that provides a dedicated private connection between your data center and a VPC.|
|[Network Access Control List (network ACL)(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)|A network ACL allows or denies specific inbound or outbound traffic at the subnet level using stateless packet filtering.|
|[Security groups(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)|Security groups control the inbound and outbound traffic for a resource at the instance level using stateful packet filtering.|
|[Domain Name System (DNS)(opens in a new tab)](https://aws.amazon.com/route53/what-is-dns/)|DNS translates human readable domain names to machine readable IP addresses (for example, 192.0.2.0).|
|[Amazon Route 53(opens in a new tab)](https://aws.amazon.com/route53/)|Route 53 is a scalable and reliable DNS web service that helps developers and businesses route end users to internet applications, whether they’re hosted in AWS or elsewhere. It also supports domain registration, health checks, and advanced traffic routing policies.|
|[Amazon CloudFront(opens in a new tab)](https://aws.amazon.com/cloudfront/)|CloudFront is a web service that speeds up distribution of your web content to your users through a worldwide network of data centers called edge locations. It securely delivers content with low latency and high transfer speeds.|
|[AWS Global Accelerator(opens in a new tab)](https://aws.amazon.com/global-accelerator/)|Global Accelerator is a networking service that helps improve the availability and performance of applications for global users by routing traffic through the AWS global network. It helps improve application availability, performance, and security.|
|[Amazon Transit Gateway(opens in a new tab)](https://aws.amazon.com/transit-gateway/)|Amazon VPC Transit Gateways is a network transit hub used to interconnect VPCs and on-premises networks.|
|[NAT Gateway(opens in a new tab)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)|Network Address Translation (NAT) gateway allows instances in a private subnet to connect with services outside your VPC. External services can't initiate a connection with those instances.|
|[API Gateway(opens in a new tab)](https://aws.amazon.com/api-gateway/)|The Amazon API Gateway is an AWS service for creating, publishing, maintaining, monitoring, and securing APIs at any scale. It handles all the tasks involved in accepting and processing up to hundreds of thousands of concurrent API calls.|