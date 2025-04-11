## Consul

I wanted to install a service discovery system to manage, well, all of the other services that exist only to manage other services on this cluster.

I have the idea of installing Authelia, then Envoy, then Consul in a chain as a replacement for Nginx. Obviously it's far more complicated than Nginx, but by now that's the point; to increase the complexity of this homelab until it collapses under its own weight. Alas poor GoldenTooth. I knew him, Gentle Reader, a cluster of infinite GPIO!

First order of business is to set up the Consul servers – leader and followers – which will occupy Bettley, Cargyll, and Dalt.