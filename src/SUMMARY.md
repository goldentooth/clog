# Summary

- [Introduction](./001_introduction.md)
- [Cluster Hardware](./002_cluster_hardware.md)
- [Meet the Nodes](./003_meet_the_nodes.md)
- [General Node Configuration](./004_node_configuration.md)
- [Cluster Roles and Responsibilities](./005_cluster_roles.md)
- [Load Balancer](./006_load_balancer.md)
- [Container Runtime](./007_container_runtime.md)
- [Networking](./008_networking.md)
- [Configuring Packages](./009_configuring_packages.md)
- [Installing Packages](./010_installing_packages.md)
- [`kubeadm init`](./011_kubeadm_init.md)
- [Bootstrapping the First Control Plane Node](./012_bootstrapping_1.md)
- [Joining the Rest of the Control Plane](./013_bootstrapping_2.md)
- [Admitting the Worker Nodes](./014_bootstrapping_3.md)
- [Where Do We Go From Here?](./015_where_do_we_go_from_here.md)
- [Installing Helm](./016_installing_helm.md)
- [Installing Argo CD](./017_installing_argocd.md)
- [The "Incubator" GitOps Application](./018_the_incubator_app_of_apps.md)
- [Prometheus Node Exporter](./019_prometheus_node_exporter.md)
- [Router BGP Configuration](./020_router_bgp_configuration.md)
- [MetalLB](./021_metallb.md)
- [Testing MetalLB](./022_testing_metallb.md)
- [Refactoring Argo CD](./023_refactoring_argocd.md)
- [Giving Argo CD a Load Balancer](./024_argocd_load_balancer.md)
- [ExternalDNS](./025_external_dns.md)
- [Killing the Incubator](./026_killing_the_incubator.md)
- [Welcome Back](./027_welcome_back.md)
- [NFS Exports](./028_nfs_exports.md)
- [Kubernetes Updates](./029_kubernetes_updates.md)
- [Fixing MetalLB](./030_fixing_metallb.md)
- [NFS Mounts](./031_nfs_mounts.md)
- [Slurm](./032_slurm.md)
- [Terraform](./033_terraform.md)
- [Dynamic DNS](./034_dynamic_dns.md)
- [Load Balancer Revisited](./035_load_balancer_revisited.md)
- [CloudFront and ACM](./036_cloudfront_and_acm.md)
- [Prometheus](./037_prometheus.md)
- [Consul](./038_consul.md)
- [Vault](./039_vault.md)
- [Envoy](./040_envoy.md)
- [Step-CA](./041_step_ca.md)

<!--
  Prometheus:
    - Blackbox Exporter
    - Other fun stuff
  Slurm apps
  Kubeflow
  MLflow
  DVC
  Consul
  Vault
  Weights & Biases
  MinIO
  PostgREST
  Grafana
  Istio
  Kiali
  OpenTelemetry
  Databases:
    - Tools:
      - OTLP/OTAP:
        - PostgreSQL
        - CockroachDB or YugabyteDB (distributed)
      - Document:
        - MongoDB:
          - Project: store experiment logs or job metadata.
      - Columnar (OLAP):
        - ClickHouse or DuckDB:
          - Ideal for analytics and fast aggregation.
          - Project: time-series dashboard for cluster metrics or experiment results.
      - Key-Value / Cache:
          - Redis: perfect for queues, parameter searches, pub/sub.
          - Etcd (already in K8s) is fun to explore directly, too.
    - Projects:
      - Same data (e.g., experiment logs) → store & query across Postgres, Mongo, ClickHouse. Compare:
        - Query expressiveness
        - Insert speed
        - Aggregation patterns
        - Build a service that translates a GraphQL or REST query into native calls on different DBs.

  Event-Based & Reactive Architectures: Useful for decoupling and analyzing system resilience.
    - Message Buses:
      - NATS: super lightweight, great for microservices.
      - Kafka: heavier but real-world. Try Confluent + KRaft mode (no ZooKeeper).
      - RabbitMQ: good for queue-based workflows.
    - Patterns to Implement:
      - Event Sourcing (append-only log of state)
      - CQRS (split reads/writes with projections)
      - Pub/Sub for async processing
    - Projects:
      - ML pipeline as an event graph: data arrives → model trains → artifact saved → inference updated.
      - Simulate failure in a message-processing chain.
      - Compare Kafka vs. NATS vs. Redis Pub/Sub in terms of latency, durability, fanout.

  Load Balancing:
    - Tools:
      - HAProxy (already have): try round-robin vs. least-connections vs. sticky.
      - Traefik or NGINX: try per-path routing or TLS termination.
      - Envoy + Consul: explore L7 routing and circuit-breaking.
    - Traffic Simulators:
      - Locust: programmable load generator (Python).
      - k6: fast scripting for load tests.
      - Boom or wrk: low-level HTTP pressure.
    - Projects:
      - Compare latency under different balancing algorithms.
      - Simulate partial outages + recovery.
      - Build a dashboard showing real-time traffic distribution across services.

  - Observability as a Design Layer: Deepen your intuition for how systems feel under load or change.
    - Tools:
      - Grafana + Prometheus
      - Loki: for log aggregation
      - Jaeger: for distributed tracing
      - Netdata: high-res system metrics
    - Projects:
      - Visualize cascading latency in service chains.
      - Trace Slurm vs. K8s job timings across stack layers.
      - Build alerts that simulate pager-duty scenarios ("CPU > 80%, node drain triggered").

  - Systems Comparison Framework: Make it all dialectical — pit systems against each other, test their affordances.
    | Area              | System 1      | System 2        | Criteria                                |
    | ----------------- | ------------- | --------------- | --------------------------------------- |
    | Job Scheduling    | K8s           | Slurm           | Startup latency, resource fairness      |
    | Storage           | Ceph          | MinIO           | Throughput, replication behavior        |
    | ML Pipelines      | MLFlow        | ZenML           | UX, integration ease                    |
    | Queuing           | Kafka         | NATS            | Message loss, latency under burst       |
    | Databases         | Postgres      | ClickHouse      |	Query expressivity vs. speed            |
    | LB Strategy       | Round Robin   | Least Latency	  | Tail latency, connection churn          |

  Projects:
    - A/B experiment simulator
      - Simulate users, route through K8s or Nomad, and track outcomes with MLFlow.
      - Explore reinforcement learning or bandit algorithms.
    - "Evolutionary Cluster"
      - Run neuroevolution or genetic algorithms across Slurm nodes.
      - Evolve tiny programs, neural nets, or even Bash scripts.
    - Distributed Hyperparameter Search
      - Simple K8s jobs that fetch param sets from Redis/Postgres and push results to MLFlow.
      - Run the same jobs under Slurm and compare orchestration overhead.
    - Chaos Testing with Litmus or Chaos Mesh
      - Introduce random node failures or network partitions.
      - Learn how K8s/Slurm/Nomad respond differently to chaos.
    - Homegrown ML Platform
      - Make a super simple "submit your model YAML here" + GitOps + MLFlow setup.

    - System Level Contrasts
      - Same workload → multiple backends.
      - ML training pipeline in DVC → Slurm vs. K8s.
      - Compare ergonomics of Kubernetes vs. Nomad vs. Slurm
      - Tiny web service (Flask, FastAPI) → deploy via Nomad, K8s, and plain systemd.

    - Sample System Design Challenges: These can be scoped small but illustrate huge ideas
      - Design a self-healing service mesh:
        - Node goes down → service migrates → traffic reroutes.
        - Use HAProxy + Consul + health checks.
      - Build an autoscaling worker pool:
        - Job queue fills → Slurm or K8s spawns compute pods → results tracked via Redis.
      - Implement GitOps + Event-Driven Dataflow:
        - DVC pushes trigger ArgoCD updates → new pipeline launches → result saved.
      - Compare consistency models:
        - Build a mini CRDT system in Redis.
        - Model eventual consistency with Redis vs. strict with Postgres.

  - System Design Projects:
    - Designing a URL Shortening Service like TinyURL
    - Designing Pastebin
    - Designing Instagram
    - Designing Dropbox
    - Designing Facebook Messenger
    - Designing Twitter
    - Designing Youtube or Netflix
    - Designing Typeahead Suggestion
    - Designing an API Rate Limiter
    - Designing Twitter Search
    - Designing a Web Crawler
    - Designing Facebook’s Newsfeed
    - Designing Yelp or Nearby Friends
    - Designing Uber backend
    - Designing Ticketmaster


-->