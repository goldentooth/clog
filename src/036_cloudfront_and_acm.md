# CloudFront and ACM

The next step will be to set up a CloudFront distribution that uses this address format as an origin, with no caching, and an ACM certificate. Assuming I can do that. If I can't, I might need to figure something else out. I could also use CloudFlare, and indeed if anyone ever reads this they're probably screaming at me, "just use CloudFlare, you idiot," but I'm trying to restrict the number of services and complications that I need to keep operational simultaneously.

Plus, I use Safari (and Brave) rather than Chrome, and one of the only systems with which I seem to encounter persistent issues using Safari is... CloudFlare. It might not for my use case, but I figure I would need to set it up just to test it.

So, yes, I'm totally aware this is a nasty hack, but... I'm gonna try it.

Spelling this out a little, here's the explicit idea:
  - Make a request to `service.home-proxy.goldentooth.net`
  - That does DNS lookup, which points to a CloudFront distribution
  - TLS certificate loads for CloudFront
  - CloudFront makes request to my home internet, preserving the Host header
  - That request gets port-forwarded to Nginx
  - Nginx matches host header `service.home-proxy.goldentooth.net` and sets `$subdomain` to `service`
  - Nginx sets upstream server name to `service.goldentooth.net`
  - Nginx does DNS lookup for upstream server and finds `10.4.11.43`
  - Nginx proxies request back to `10.4.11.43`

And this appears to work:

```
$ curl https://httpbin.home-proxy.goldentooth.net/ip
{
  "origin": "66.61.26.32"
}
```

The latency is nonzero but not noticeable to me. It's still an ugly hack, and there are some security implications I'll need to deal with.
