# Loki

_This, the previous "article" (on [Grafana](./043_grafana.md)), and the next one (on [Vector](./045_vector.md)), are occurring mostly in parallel so that I can validate these services as I go._

Loki is... there's a whole lot going on there. My only real action at present was to enable a retention policy so that my logs wouldn't grow without bound until the end of time. This coincided with me noticing that my `/var/log/journal` directories had gotten up to about 4GB, which led me to perform a similar change in the `journald` configuration.
