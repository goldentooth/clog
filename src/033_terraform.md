# Terraform

I would like my Goldentooth services to be reachable from anywhere, but 1) I'd like trouble-free TLS encryption to the domain, and 2) I'd like to hide my home IP address. I can set up LetsEncrypt, but I've had issues in the past with local storage of certificates on what is essentially an ephemeral setup that might get blown up at any point. I'd like to prevent spurious strain on their systems, outages due to me spamming, etc etc etc. I'd prefer it if I could use AWS ACM. If I can use ACM and CloudFront and just not cache anything, or cache intelligently on a per-domain basis, that would be nice. I don't know if I can get that working - I know AWS intends ACM for AWS services – but I'll try.

So the first step of this is to set up Terraform; to create an S3 bucket to hold the state and a lock to support state locking.

We can bootstrap this by just creating the S3 bucket, then creating a Terraform configuration that only contains that S3 bucket and imports the existing bucket (mostly so I don't forget what the bucket is for or what it is using). I apply that - yup, works.

The next thing I add is configuration for an OIDC provider for GitHub. Fortunately, there's a [provider](https://registry.terraform.io/modules/terraform-module/github-oidc-provider/aws/latest) for this, so it's easy to set up. I apply that and it creates an IAM role. I assign it Administrator access temporarily.

I create a GitHub Actions workflow to set up Terraform, plan, and apply the configuration. That works when I push to `main`. Pretty sweet.
