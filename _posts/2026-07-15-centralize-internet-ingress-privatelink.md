---
layout: post
title:  "Centralize internet ingress for isolated workloads with AWS PrivateLink"
date:   2026-07-15
author: prashant
categories: [blog]
tags: [aws, privatelink, networking, cloudfront, vpc]
excerpt_separator: <!--more-->
---

In many organizations, application teams own their workloads and a central networking team owns the path from the internet. Application teams want to deploy without waiting on network changes. The networking team wants one place to apply edge protection, web filtering, and traffic inspection. And security teams want workloads that cannot reach the internet at all.

<!--more-->

In this post, I show how to meet all three goals with a shared ingress virtual private cloud (VPC) that delivers traffic to isolated workload VPCs through [AWS PrivateLink](https://aws.amazon.com/privatelink/). The networking team runs an Application Load Balancer (ALB) in the ingress VPC and, optionally, Amazon CloudFront, AWS WAF, and AWS Network Firewall. Each application team runs its own VPC with no internet gateway and no NAT gateway, and publishes its application as a PrivateLink endpoint service. You deploy everything with modular AWS CloudFormation templates and the AWS Command Line Interface (AWS CLI), and a stub application shows you, request by request, that the traffic took the path you expect.

This pattern builds on [How to securely publish internet applications at scale using Application Load Balancer and AWS PrivateLink](https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-securely-publish-internet-applications-at-scale-using-application-load-balancer-and-aws-privatelink/). This post adds CloudFront VPC origins, Network Firewall inspection inside the ingress VPC, a self-service split between the networking and application teams, and guidance on when you need each layer.

## Solution overview

Figure 1 shows the architecture.

![Architecture diagram]({{ site.baseurl }}/assets/images/2026/centralize-internet-ingress-privatelink/architecture.png)

*Figure 1: Centralized ingress VPC delivering traffic to an isolated workload VPC through AWS PrivateLink*

A request travels through the following components:

1. **CloudFront (optional)** terminates the viewer connection at the edge. An AWS WAF web ACL with CloudFront scope can filter requests there. CloudFront reaches the ALB through a [VPC origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html), so the ALB is internal and has no public IP address.
2. **The ALB** in the ingress VPC routes the request. Without CloudFront, the ALB is internet facing and a Regional AWS WAF web ACL protects it.
3. **Network Firewall (optional)** inspects the traffic between the ALB subnets and the endpoint subnets.
4. **An interface VPC endpoint** in the ingress VPC is the PrivateLink consumer. The ALB targets the IP addresses of its network interfaces.
5. **A Network Load Balancer (NLB) and endpoint service** in the workload VPC are the PrivateLink provider. The NLB forwards the request to the application.

AWS WAF is not a separate hop in this path. It is attached to the CloudFront distribution or to the ALB and evaluates requests there.

### Why PrivateLink

PrivateLink was built for service providers, and three of its properties make it a good fit here:

- **Connections are one way.** Only the consumer (the ingress VPC) can start a connection. The workload cannot initiate connections back into the ingress VPC through the endpoint.
- **IP address ranges can overlap.** PrivateLink translates addresses on both sides. In this post, both VPCs use `10.0.0.0/16` on purpose. The application sees requests coming from private IP addresses of its own NLB, not from the ingress VPC.
- **Traffic stays on the AWS network** between the endpoint and the endpoint service.

Because each application gets its own endpoint, ALB, and (optionally) CloudFront distribution, applications scale and change independently, while one team manages all of them in one VPC.

### Who owns what

The solution has four CloudFormation templates. The teams hand values to each other through AWS Systems Manager Parameter Store, so nobody copies IDs between stacks.

| Template | Owner | Deployed | Creates |
|---|---|---|---|
| `ingress-core.yaml` | Networking | Once | Ingress VPC, subnets in two Availability Zones, internet gateway, route tables, optional Network Firewall and logging. Publishes `/pattern/ingress/*` parameters. |
| `waf.yaml` | Networking | Once per scope | AWS WAF web ACL with a test rule, a rate-based rule, and the AWS managed core rule set. |
| `workload.yaml` | Application | Once per application | Workload VPC with no internet access, stub application on Amazon EC2, NLB, and endpoint service. Publishes `/pattern/apps/<AppName>/endpoint-service-name`. |
| `ingress-app.yaml` | Networking | Once per application | Interface endpoint, ALB, the custom resource that registers the endpoint IP addresses, and the optional CloudFront distribution. |

The ingress core publishes Availability Zone IDs (for example, `usw2-az1`), not Availability Zone names. An interface endpoint can only use Availability Zones where the endpoint service exists, and Availability Zone names map to different physical zones in different accounts.

## Prerequisites

- An AWS account and permissions to create the resources in the templates, including IAM roles.
- The [AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), configured for the Region you use. I tested in US West (Oregon), `us-west-2`. Use a Region where [CloudFront VPC origins are supported](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html) if you want the CloudFront option, and choose Availability Zone IDs that support it.
- `bash` and `curl`.
- The templates and scripts from the accompanying repository: [https://github.com/opstodevops/privatelink-centralized-ingress](https://github.com/opstodevops/privatelink-centralized-ingress).

## Walkthrough

You can deploy everything with one script, or run the stacks one at a time to see what each team does. The script is a thin wrapper around `aws cloudformation deploy`.

```bash
# Core path only: internet-facing ALB, no CloudFront, no firewall
./scripts/deploy.sh

# Full path: CloudFront VPC origin, AWS WAF, Network Firewall, and logging
./scripts/deploy.sh --cloudfront --waf --firewall --logging
```

In my tests, the full path took about 26 minutes, most of it for the Network Firewall endpoints and the CloudFront distribution. The core path took about 7 minutes.

The following steps show the core path stack by stack.

### Step 1: The networking team deploys the ingress core

```bash
aws cloudformation deploy \
  --stack-name pattern-ingress-core \
  --template-file templates/ingress-core.yaml \
  --parameter-overrides EnableCloudFront=false EnableNetworkFirewall=false
```

Choose `EnableCloudFront` before you deploy the first application. With `true`, the ALB subnets have no route to the internet gateway, and CloudFront is the only way in. The VPC still needs an internet gateway attached, because [CloudFront VPC origins require one](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html), even though no traffic is routed through it.

Check the values the stack published:

```bash
aws ssm get-parameters-by-path --path /pattern/ingress --recursive \
  --query 'Parameters[].[Name,Value]' --output text
```

### Step 2: The application team deploys the workload

```bash
aws cloudformation deploy \
  --stack-name pattern-workload-app1 \
  --template-file templates/workload.yaml \
  --parameter-overrides AppName=app1
```

The application team needs only an application name. The template reads the allowed principal and the Availability Zone IDs from Parameter Store through parameters of type `AWS::SSM::Parameter::Value<String>`.

The workload VPC has no internet gateway, no NAT gateway, and no default route. The stub application is a small Python HTTP server on Amazon Linux 2023, written by user data, because the instance cannot download anything.

The endpoint service allows only the ingress account to connect, and it accepts connections from that principal automatically:

```yaml
  EndpointService:
    Type: AWS::EC2::VPCEndpointService
    Properties:
      NetworkLoadBalancerArns: [!Ref Nlb]
      AcceptanceRequired: false

  EndpointServicePermissions:
    Type: AWS::EC2::VPCEndpointServicePermissions
    Properties:
      ServiceId: !Ref EndpointService
      AllowedPrincipals:
        - !Ref IngressPrincipalArn
```

Granting the permission to one specific principal is what makes automatic acceptance safe. If you allow a broad set of principals, set `AcceptanceRequired: true` and approve each connection.

### Step 3: The networking team connects the application

```bash
aws cloudformation deploy \
  --stack-name pattern-ingress-app1 \
  --template-file templates/ingress-app.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides AppName=app1 \
    AllowedCidr="$(curl -s https://checkip.amazonaws.com)/32"
```

The template reads the endpoint service name with a dynamic reference, `{% raw %}{{resolve:ssm:/pattern/apps/${AppName}/endpoint-service-name}}{% endraw %}`, and creates the interface endpoint with private DNS turned off.

An ALB can't target a VPC endpoint directly, but it can target IP addresses. A small AWS Lambda-backed custom resource looks up the endpoint's network interfaces and registers their IP addresses in the ALB target group:

```python
enis = ec2.describe_network_interfaces(
    NetworkInterfaceIds=props["NetworkInterfaceIds"])["NetworkInterfaces"]
wanted = {e["PrivateIpAddress"] for e in enis}
current = {t["Target"]["Id"] for t in
           elb.describe_target_health(TargetGroupArn=tg)["TargetHealthDescriptions"]}
if wanted - current:
    elb.register_targets(TargetGroupArn=tg,
                         Targets=[{"Id": ip, "Port": 80} for ip in wanted - current])
```

The function runs when the stack is created or updated. If you change the endpoint subnets, update the stack so that the target group follows.

### Step 4: Verify the path

Send a request to the URL in the stack output. The endpoint IP addresses can take a minute or two after the stack completes to pass ALB health checks. Until then, some requests can time out.

```bash
URL=$(aws cloudformation describe-stacks --stack-name pattern-ingress-app1 \
  --query "Stacks[0].Outputs[?OutputKey=='Url'].OutputValue" --output text)
curl -s "$URL"
```

The stub application describes what it saw. The following response is from my deployment with CloudFront enabled:

```json
{
  "app": "app1",
  "served_by": {
    "instance_id": "i-0123456789abcdef0",
    "az": "us-west-2b",
    "private_ip": "10.0.0.174"
  },
  "client_source_ip_seen_by_app": "10.0.0.165",
  "via_cloudfront": true,
  "x_forwarded_for": "198.51.100.25, 130.176.72.8",
  "host": "internal-pattern--alb-c8jutltujuvw-979464256.us-west-2.elb.amazonaws.com",
  "outbound_internet_blocked": true
}
```

Each field confirms part of the design:

- `client_source_ip_seen_by_app` is an address of the workload NLB, inside the workload's own `10.0.0.0/16`. The overlapping ranges don't matter, because PrivateLink translates addresses.
- `x_forwarded_for` carries the original client address, followed by the CloudFront address, so the application can still log and authorize by client IP.
- `via_cloudfront` is `true` because the request carries the `X-Amz-Cf-Id` header.
- `outbound_internet_blocked` is `true` because the instance could not open a connection to a public IP address within two seconds.

## Add the optional layers

### CloudFront with a VPC origin

With `--cloudfront`, `ingress-app.yaml` makes the ALB internal and adds an `AWS::CloudFront::VpcOrigin` and a distribution. The ALB security group allows inbound traffic only from the AWS managed prefix list `com.amazonaws.global.cloudfront.origin-facing`. CloudFormation can't look up a prefix list ID by name, so the script looks it up and passes it as a parameter:

```bash
aws ec2 describe-managed-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing \
  --query 'PrefixLists[0].PrefixListId' --output text
```

The distribution uses the managed `CachingDisabled`, `AllViewerExceptHostHeader`, and `SecurityHeadersPolicy` policies. It uses the default `*.cloudfront.net` certificate and HTTP between CloudFront and the ALB, which is fine for testing but not for production. For production, use your own domain, an AWS Certificate Manager (ACM) certificate on the distribution and on the ALB, and set the VPC origin protocol policy to `https-only`.

Requests sent directly to the ALB DNS name now fail, because the ALB has no public address.

Creating the VPC origin and distribution took about 15 minutes in my tests. Decide on CloudFront before you deploy an application. Switching an existing `ingress-app` stack between an internet-facing and an internal ALB replaces the load balancer, and in my tests that update failed and rolled back. Delete the stack and create it again instead.

### AWS WAF

`waf.yaml` creates a web ACL with three rules: a test rule that returns `403` for any `User-Agent` that contains `badbot`, a rate-based rule per client IP address, and the `AWSManagedRulesCommonRuleSet` managed rule group. Web ACLs for CloudFront must be created in `us-east-1`, so the script deploys a `CLOUDFRONT` web ACL there when CloudFront is enabled, and a `REGIONAL` web ACL in your Region otherwise.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -A "badbot" "$URL"
# 403
```

### Network Firewall inspection in the ingress VPC

With `--firewall`, `ingress-core.yaml` creates one firewall per Availability Zone, each with its endpoint in a dedicated subnet and all sharing one firewall policy, and adds routes so that traffic between each ALB subnet and the endpoint subnet in the same Availability Zone goes through that zone's firewall endpoint in both directions. These routes are more specific than the VPC's local route, which is supported by [VPC routing enhancements](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall-with-vpc-routing-enhancements/).

The firewall policy uses strict rule order. It includes the [active threat defense](https://docs.aws.amazon.com/network-firewall/latest/developerguide/aws-managed-rule-groups-atd.html) managed rule group `AttackInfrastructureStrictOrder`, and a sample rule group that rejects HTTP requests whose path starts with `/blocked`:

```text
reject http any any -> any any (msg:"Blocked URI /blocked"; flow:to_server,established; http.uri; content:"/blocked"; startswith; sid:1000001; rev:1;)
```

Active threat defense is on by default only for firewall policies created in the console. If you create policies with CloudFormation or the API, add the managed rule group yourself, as this template does.

Inserting a firewall between two subnets has two consequences that the template handles:

- **Security group references stop working across the firewall.** The endpoint security group can't use the ALB security group as its source. With inspection turned on, the endpoint security group allows the ALB subnet CIDR ranges instead.
- **Traffic must stay symmetric.** An ALB node must use only the endpoint IP address in its own Availability Zone, so the request and the response pass through the same firewall endpoint. The template turns off cross-zone load balancing on the ALB target group when inspection is on.

To test the firewall, request a blocked path:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "${URL}blocked"
# 502
```

The ALB returns `502` because the firewall resets the connection. A `drop` rule would make the request hang until it timed out instead. The alert log in Amazon CloudWatch Logs shows the ALB node as the source, the endpoint network interface as the destination, and the original client in the `xff` field:

```json
{"src_ip":"10.0.0.192","dest_ip":"10.0.2.179",
 "alert":{"signature":"Blocked URI /blocked","action":"blocked"},
 "http":{"url":"/blocked","xff":"198.51.100.25"}}
```

Use the firewall alert logs, flow logs, and CloudWatch metrics to check inspection. In my tests, VPC Reachability Analyzer reported no path between the ALB and the endpoint once the firewall routes were in place, even though traffic flowed correctly.

## Which layers do you need?

You don't need every layer for every application. Use the following as a starting point.

| Requirement | Recommended option |
|---|---|
| You want CloudFront in front of private ALBs or NLBs in other accounts, and you don't need a central inspection point | [Cross-account CloudFront VPC origins](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-cross-account-support-for-amazon-cloudfront-virtual-private-cloud-vpc-origins/) shared with AWS Resource Access Manager (AWS RAM). No ingress VPC is needed. |
| One team must own all internet-facing load balancers, workloads must have no internet path at all, or the VPC ranges overlap | Ingress VPC with ALB and PrivateLink (the core path in this post), with CloudFront and AWS WAF as needed. |
| You also need network-level inspection, such as Suricata rules, threat intelligence, or protocol checks beyond HTTP request attributes | Add Network Firewall in the ingress VPC. |

AWS WAF evaluates HTTP requests, and Network Firewall inspects network flows and can apply Suricata-compatible rules. They complement each other. If AWS WAF covers your filtering needs, you can skip Network Firewall and its hourly and per-GB charges.

## Going multi-account

To keep the steps short, this post deploys both VPCs in one account. To run the workload in its own account, change the following:

- **Allowed principal.** The ingress core already publishes the networking account root (`arn:aws:iam::<account-id>:root`) as the allowed principal. To narrow it, publish a specific IAM role ARN instead.
- **Parameter handoff.** CloudFormation can't resolve [Systems Manager parameters shared from another account](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references-ssm.html). The networking team can write the `/pattern/ingress/*` parameters into each workload account with a service-managed CloudFormation StackSet targeted at the workload organizational unit. In the other direction, the application team gives the endpoint service name to the networking team. Because `ingress-app.yaml` reads the name with a same-account dynamic reference, change it to a plain template parameter and pass the name in when you deploy.
- **Availability Zones.** Keep using Availability Zone IDs. Availability Zone names differ between accounts.

## Cost

The solution uses billed resources, including the ALB, NLB, interface endpoints, an EC2 instance, and, if you turn them on, Network Firewall endpoints, active threat defense data processing, AWS WAF web ACLs and rules, CloudFront requests, and CloudWatch Logs. Network Firewall endpoints are billed per hour in each Availability Zone, and are the largest fixed cost in the full deployment. Use the [AWS Pricing Calculator](https://calculator.aws/) to estimate cost for your Region and traffic, and delete the resources when you finish testing.

## Clean up

Delete the stacks in reverse order. An endpoint service can't be deleted while an endpoint is connected to it, so the script deletes the per-application stacks first:

```bash
./scripts/cleanup.sh app1
```

Cleanup of the full deployment took about 18 minutes in my tests. Afterward, confirm that no `pattern-*` stacks remain in your Region and in `us-east-1`.

## Conclusion

In this post, you deployed a shared ingress VPC that a networking team manages and that delivers internet traffic to an isolated workload VPC through AWS PrivateLink. The workload has no path to the internet, its IP range can overlap with the ingress VPC, and the application team needs only an application name to publish its service. You added CloudFront VPC origins, AWS WAF, and Network Firewall as optional layers, and verified each one with the stub application and the service logs.

As next steps, replace the stub with your own application behind the NLB, add HTTPS with your own domain, and automate the per-application `ingress-app` deployment for new applications. To learn more, see the [AWS PrivateLink documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/), the [AWS Network Firewall deployment models](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/), and [centralized ingress in the multi-VPC whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-network-firewall-for-centralized-ingress.html).

---

*About the author* (to be added)
