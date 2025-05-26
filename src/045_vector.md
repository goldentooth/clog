# Vector

_This and the two previous "articles" (on [Grafana](./043_grafana.md) and on [Vector](./045_vector.md)) are occurring mostly in parallel so that I can validate these services as I go._

The main thing I wanted to do immediately with Vector was hook up more sources. A couple were turnkey (`journald`, `kubernetes_logs`, `internal_logs`) but most were just log files. These latter are not currently parsed according to any specific format, so I'll need to revisit this and extract as much information as possible from each.

It would also be good for me to inject some more fields into this that are set on a per-node level. I already have hostname, but I should probably inject IP address, etc, and anything else I can think of.

Other than that, it doesn't really seem like there's a lot to discuss here. Vector's cool, though. And in the future, I should remember that adding a whole bunch of log files into Vector from ten nodes, all at once, is not a great idea, as it will flood the Loki sink...
