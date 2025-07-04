# Loki

_This, the previous "article" (on [Grafana](./043_grafana.md)), and the next one (on [Vector](./045_vector.md)), are occurring mostly in parallel so that I can validate these services as I go._

Loki is... there's a whole lot going on there.

I enabled a retention policy so that my logs wouldn't grow without bound until the end of time. This coincided with me noticing that my `/var/log/journal` directories had gotten up to about 4GB, which led me to perform a similar change in the `journald` configuration.

I reduced the `retention_delete_worker_count` from 150 to 5 🙂

I also configured Loki to use Consul as its ring kvstore, which involved sketching out an ACL policy and generating a token, but nothing too weird. (Assuming that it works.)

The next several hours involved cleanup after I rashly configured Loki to use TLS. I didn't know that I'd then need to configure Loki to communicate with itself via TLS, and that I would have to do so in several different places and that those places would have different syntax for declaring the same core ideas (CA cert, TLS cert, TLS key).

And holy crap, the Loki site is absolutely awful for finding and understanding where some configuration is needed.
