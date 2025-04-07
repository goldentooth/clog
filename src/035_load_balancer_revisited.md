# Load Balancer Revisited

As [previously stated](./033_terraform.md): I would like my GoldenTooth services to be reachable from anywhere, but 1) I'd like trouble-free TLS encryption to the domain, and 2) I'd like to hide my home IP address. I can set up LetsEncrypt, but I've had issues in the past with local storage of certificates on what is essentially an ephemeral setup that might get blown up at any point. I'd like to prevent spurious strain on their systems, outages due to me spamming, etc etc etc. I'd prefer it if I could use AWS ACM. If I can use ACM and CloudFront and just not cache anything, or cache intelligently on a per-domain basis, that would be nice. I don't know if I can get that working - I know AWS intends ACM for AWS services – but I'll try.

Now, one thing I want to be able to do for this is to have a single origin for the CloudFront distribution, e.g. *.my-home.goldentooth.net, which will resolve to my home IP address. But I want to be able to route based on domain name. I already have `<service>.goldentooth.net` working with [ExternalDNS](./025_external_dns.md) and [MetalLB](./021_metallb.md). So I want my reverse proxy to map an incoming request for `<service>.my-home.goldentooth.net` to a backend `<service>.goldentooth.net` with as little extra work as possible. Performance is less of an issue here than the fact that it works, that it's easy to maintain and repair if it breaks three year from now, and that I can complete this and move on to something else.

These factors combined mean that I should not use HAProxy for this. HAProxy is incredibly powerful and very performant, but it is not incredibly flexible for this sort of ad-hoc YOLO kind of work. Nginx, however, is.

So, alongside HAProxy, which I'm using for Kubernetes high-availability, I'll open a port on my router and forward it to Nginx, which will reverse-proxy that based on the domain name to the appropriate local load balancer service.

The resulting configuration is pretty simple:

```nginx
server {
  listen 8080;
  resolver 8.8.8.8 valid=10s;
  server_name ~^(?<subdomain>[^.]+)\.{{ cluster.cloudfront_origin_domain }}$;
  location / {
    set $target_host "$subdomain.{{ cluster.domain }}";
    proxy_pass http://$target_host;
    proxy_set_header Host $target_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_ssl_verify off;
  }
}
```

And it just works; requesting http://httpbin.my-home.goldentooth.net:7463/ returns the appropriate service.

The next step will be to set up a CloudFront distribution that uses this address format as an origin, with no caching, and an ACM certificate. Assuming I can do that. If I can't, I might need to figure something else out. I could also use CloudFlare, and indeed if anyone ever reads this they're probably screaming at me, "just use CloudFlare, you idiot," but I'm trying to restrict the number of services and complications that I need to keep operational simultaneously.

Plus, I use Safari (and Brave) rather than Chrome, and one of the only systems with which I seem to encounter persistent issues using Safari is... CloudFlare. It might not for my use case, but I figure I would need to set it up just to test it.

So, yes, I'm totally aware this is a nasty hack, but... I'm gonna try it.
