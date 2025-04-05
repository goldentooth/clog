# Installing Helm

I have a lot of ambitions for this cluster, but after some deliberation, the thing I most want to do right now is deploy something _to_ Kubernetes.

So I'll be starting out by installing Argo CD, and I'll do that... soon. In the next chapter. I decided to install Argo CD via Helm, since I expect that Helm will be useful in other situations as well, e.g. trying out applications before I commit (no pun intended) to bringing them into GitOps.

So I created a [playbook](https://github.com/goldentooth/cluster/blob/main/playbooks/install_helm.yaml) and [role](https://github.com/goldentooth/cluster/tree/main/roles/goldentooth.install_helm) to cover installing Helm.

Fortunately, this is fairly simple to install and trivial to configure, which is not something I can say for Argo CD 🙂
