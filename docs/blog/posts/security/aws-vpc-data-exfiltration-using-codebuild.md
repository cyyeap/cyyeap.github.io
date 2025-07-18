---
layout: post
title: AWS VPC data exfiltration using CodeBuild
date: 2022-02-03T04:48:52
categories:
  - AWS
---

**UPDATE**: This issue has been fixed. See the end of the article for timeline
and details.

In September 2020, I published a [guest blog post][ian-blog] on [Ian Mckay][ian]'s
blog. The tl;dr is that "escaping" a privileged container running in an Amazon-managed
AWS account isn't a security concern for Amazon, thanks to defence-in-depth on
both an EC2 and IAM level. Here's a few paragraphs I left out of that blog post
at the time.

## But wait, there's more

As Corey said in the tweet at the start of that story, you _can_ grant CodeBuild
access to resources in your own VPC. CodeBuild is much like AWS Lambda in that 
regard: by default it has public Internet connectivity *or* you can allow it to
attach to your VPC to access internal resources. 

A [tweet by @amcabee13][anna-tweet] on her favourite topic of IP addresses for
VPC-unattached Lambdas got me thinking: some people attach Lambdas to VPCs 
because then it allows more observability and control over data egress, i.e. 
connections out to the Internet. The same goes for CodeBuild jobs: traffic will
now reach the Internet through whatever egress path you have and the associated 
security controls (firewalls, etc). But what about the EC2 instance the CodeBuild
container is running on - it's still in the AWS-managed VPC?

So naturally, I went back and checked. Running commands in the container had 
connectivity to the VPC. And running them on the host had connectivity directly 
to the public Internet, unassociated with the customer's (my) VPC. Could this 
make for very convenient data exfiltration? The answer is: yes, quite easy! 

First, run commands like these on the CodeBuild **host**:

```
ssh-keygen
cat ~/.ssh/id_rsa.pub > /home/ec2-user/.ssh/authorized_keys
ssh -f -N -D 169.254.170.8:1080 ec2-user@localhost
```

Now in the CodeBuild **container** you can run:

```
# this is routed through the customer VPC
curl http://10.1.2.3/internal 

# this is routed through the AWS VPC, bypassing your VPC egress routing tables
curl --socks4a 169.254.170.8:1080 https://google.com  
```

Pay careful attention to the `a` at the end of that `--socks4a` flag. If you 
instead pass `--socks4` everything will work, but the DNS resolution of `google.com` 
will be done by the container and your malicious DNS queries could be recorded 
by [Route 53 Resolver Query Logs][query-logs] in the customer VPC, not the AWS VPC.

## What would be better?

The CodeBuild host EC2 instance seems to only need Internet connectivity in 
order to connect to `codebuild.{region}.amazonaws.com` and associated services.
Instead of using a NAT gateway, perhaps the AWS-managed VPC could use interface
VPC endpoints. That would eliminate the issue entirely.

That said, do I consider this a huge issue? Not really. CodeBuild builds are
logged in CloudTrail and CodeBuild itself, so forensics are easy enough. It's
unlikely that this is going to be a problem in reality. But I found it fun and
worth sharing.

## Update

I reported this to the AWS Security team on February 4th. My initial email was 
acknowledged by a human within 6 hours. On the 24th of February I got an update 
with this (emphasis my own):

> CodeBuild executes customer builds in containers which run on single-tenanted 
> Amazon EC2 instances within a service-managed VPC.  When customers configure 
> a CodeBuild project with their VPC, CodeBuild's build container will apply 
> the same network routing rules as defined in the customer's VPC Security Group. 
> **We have updated the CodeBuild service to block all outbound network access 
> for newly created CodeBuild projects, which contain a customer-defined VPC 
> configuration**. 
> 
> CodeBuild's security isolation is enforced at the AWS account level and EC2 
> instances are never shared across AWS accounts. CodeBuild also logs all build 
> requests in CloudTrail for auditing usage within an AWS account.

I verified that the issue in this article is no longer applicable for new CodeBuild
projects. `codebuild.{region}.amazonaws.com` and other host-level dependencies
use VPC endpoints and there is no connectivity to the public Internet for VPC-attached
projects.

Honestly, I'm delighted with the outcome. Making this kind of change to a service
as well-established as AWS CodeBuild likely isn't straight forward, but the AWS
security team and the CodeBuild team managed to identify, develop and roll out 
the ideal fix in fewer than three weeks. Kudos to them.

That said, I did notice this new caveat appear in the [CodeBuild VPC docs][vpc-docs]
and I feel bad for anyone whose builds are slower now! Hopefully it's only a
minor increase.

![caveat in docs](/assets/2022-02-03-vpc-caveat.png)

[ian-blog]: https://onecloudplease.com/blog/security-september-escaping-codebuild
[ian]: https://twitter.com/iann0036
[anna-tweet]: https://twitter.com/amcabee13/status/1300541996738715649
[query-logs]: https://aws.amazon.com/blogs/aws/log-your-vpc-dns-queries-with-route-53-resolver-query-logs/
[vpc-docs]: https://docs.aws.amazon.com/codebuild/latest/userguide/vpc-support.html
