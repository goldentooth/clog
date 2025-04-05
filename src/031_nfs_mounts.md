# NFS Mounts

Now that Kubernetes is _kinda_ squared away, I'm going to set up NFS mounts on the cluster nodes.

For the sake of simplicity, I'll just set up the mounts on every node, including the load balancer (which is currently exporting the share).

This in itself isn't too complicated, but I created two template files (one for a .mount service, another for a .automount service), fought with the variables for a bit, and it seems to work.
