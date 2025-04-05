# Fixing MetalLB

As mentioned [here](./027_welcome_back.md), I purchased a new router to replace a power-hungry Dell server running OPNsense, and that cost me BGP support. This kills my MetalLB configuration, so I need to switch it to use Layer 2.

That shouldn't be too bad.

I think it's just a matter of deleting the BGP advertisement:

```bash
$ sudo kubectl -n metallb delete BGPAdvertisement primary
bgpadvertisement.metallb.io "primary" deleted
```

and creating an L2 advertisement:

```bash
$ cat tmp.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary
  namespace: metallb

$ sudo kubectl apply -f tmp.yaml
l2advertisement.metallb.io/primary created
```

After adding the static route to my router, I can see the friendly `go-httpbin` response when I navigate to https://10.4.11.1/

I also lost some control over DNS, e.g. the router's DNS server will override all lookups for hellholt.net rather than forwarding requests to my DNS servers.

So I created a new domain, goldentooth.net, to handle this cluster. A couple of tweaks to ExternalDNS and some service definitions and I can verify that ExternalDNS is setting the DNS records correctly, although I don't seem to be able to resolve names just yet.

I think I still need to get TLS working too, but I've soured on the idea of maintaining a cert per domain name and per service. I think I'll just have a wildcard over goldentooth.net and share that out. Too much aggravation otherwise. That's a problem for another time, though.
