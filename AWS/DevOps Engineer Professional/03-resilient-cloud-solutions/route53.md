# Amazon Route 53

- It is a highly available, scalable, fully managed, and *authoritative* DNS
    - Authoritative: the customer can update the DNS records
- Route 53 is also a Domain Registrar
- We have the ability to check the health of our resources for which we create DNS records
- It is the only AWS service which provides 100% availability SLA

## DNS Records

- Each record contains:
    - Domain/subdomain name: example com
    - Record type: `A`, `AAAA`, `CNAME`, etc.
    - Value: for example an IP address
    - Routing Policy: how Route 53 responds to queries
    - TTL: amount of time the record is cached at the DNS server
- Route 53 supports a bunch of record types:
    - `A` / `AAAA`/ `CNAME` / `NS`
    - Advanced records: `CAA` / `DS` / `MX` / `NAPTR` / `PTR` / `SOA` / `TXT` / `SPF` / `SRV`
- Important record types (from the exam perspective):
    - `A`: maps a hostname to an IPv4 address
    - `AAAA` : maps a hostname to an IPv6 address
    - `CNAME`: maps a hostname to another hostname:
        - The target is a domain name which must have an `A` or `AAAA` record
        - We can't create a `CNAME` record for the top node of a DNS namespace (Zone Apex)
    - `NS`: Name Servers for the Hosted Zone

## Hosted Zones

- Hosted Zones are containers for records that define how to route traffic to a domain and its subdomains
- Public Hosted Zones: contain records that specify how to route traffic on the internet
- Private Hosted Zones: contain records that can be resolved within one or more VPCs (private domain names)
- We pay $0.5 per month for any hosted zone we create

## Routing Policies

- [Simple routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-simple.html)
    
- [Failover routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
    
- [Geolocation routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
    
- [Geoproximity routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html)
    
- [Latency-based routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
    
- [IP-based routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-ipbased.html)
    
- [Multivalue answer routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-multivalue.html)
    
- [Weighted routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)

### Weighted

- We control the % of the requests that go to each specific resource
- We assign each record with a relative weight:
    - Weights don't need to sum up to 100%
- DNS records must have the same name and type
- They can be associated with health checks
- Use cases: load balancing between regions, testing new application versions, etc.
- We can assign a weight of 0 to a record to stop sending traffic to the resource
- If all the records have a weight of 0, then all the resources will be resolved equally

### Latency-based

- The hosted zone will resolve the resource that has the least latency for the requestor
- Helpful when latency is a priority for the users
- Latency is based on traffic between users and AWS Regions
- Records can be associated with health checks

## Failover (Active-Passive)

- With a Failover record, Route 53 actively returns a primary resource
- If there is a failure for the primary resource, the secondary resource will be returned
- When creating a Failover record, we have to select if the record is **primary** or **secondary**