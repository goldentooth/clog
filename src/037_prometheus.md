# Prometheus

Way back in [Chapter 19](./019_prometheus_node_exporter.md), I set up an Prometheus Node Exporter "app" for Argo CD, but I never actually set up Prometheus itself.

That's really fairly odd for me, since I'm normally super twitchy about metrics, logging, and observability. I guess I put it off because I was dealing with some kind of existential questions; where would Prometheus live, how would it communicate, etc, but then ended up kinda running out of steam before I answered the questions.

So, better late than never, I'm going to work on setting up Prometheus in a nice, decentralized kind of way.

First, it does appear that there's an [official role](https://prometheus-community.github.io/ansible/branch/main/prometheus_role.html) to install and configure Prometheus, so I think I'll attempt to use that rather than rolling my own. The depth to Prometheus is, after all, configuring and using it, not merely in installing it.

I also configured HAProxy (which has a Prometheus exporter built in) and the Nginx exporter. The Kubernetes nodes already have the Node Exporter configured, but the load balancer node did not, so I configured it there. The scrape configuration was fairly straightforward.
