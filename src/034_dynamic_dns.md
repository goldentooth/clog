# Dynamic DNS

As [previously stated](./033_terraform.md): I would like my Goldentooth services to be reachable from anywhere, but 1) I'd like trouble-free TLS encryption to the domain, and 2) I'd like to hide my home IP address. I can set up LetsEncrypt, but I've had issues in the past with local storage of certificates on what is essentially an ephemeral setup that might get blown up at any point. I'd like to prevent spurious strain on their systems, outages due to me spamming, etc etc etc. I'd prefer it if I could use AWS ACM. If I can use ACM and CloudFront and just not cache anything, or cache intelligently on a per-domain basis, that would be nice. I don't know if I can get that working - I know AWS intends ACM for AWS services – but I'll try.

The next step of this is to get my router to update Route53 with my home IP address whenever it changes. That's going to require a Lambda function, API Gateway, an SSM Parameter for the credentials, an IAM role, etc. That's all going to be deployed and managed via Terraform.

![dynamic-dns graph](034_dynamic_dns_graphviz.svg)
