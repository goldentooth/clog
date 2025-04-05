# Giving Argo CD a Load Balancer

All this time, the Argo CD server has been operating with a ClusterIP service, and I've been manually port forwarding it via `kubectl` to be able to show all of these beautiful screenshots of the web UI.

That's annoying and we don't have to do it anymore. Fortunately, it's very easy to change this now; all we need to do is modify the Helm release values slightly; change `server.service.type` from 'ClusterIP' to 'LoadBalancer' and redeploy. A few minutes later, we can access Argo CD via http://10.4.11.1, no port forwarding required.
