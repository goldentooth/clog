# TLS Certificate Renewal

So [some time back](./041_step_ca.md) I configured `step-ca` to generate TLS certificates for various services, but I gave the certs very short lifetimes and didn't set up renewal, so... whenever I step away from the cluster for a few days, everything breaks 🙃

Today's goal is to fix that.

```
$ consul members
Error retrieving members: Get "http://127.0.0.1:8500/v1/agent/members?segment=_all": dial tcp 127.0.0.1:8500: connect: connection refused
```

Indeed, very little is working.

Fortunately, `step-ca` provides [good instructions](https://smallstep.com/docs/step-ca/renewal/) for dealing with this sort of situation. I created a `cert-renewer@service` file:

```conf
[Unit]
Description=Certificate renewer for %I
After=network-online.target
Documentation=https://smallstep.com/docs/step-ca/certificate-authority-server-production
StartLimitIntervalSec=0
; PartOf=cert-renewer.target

[Service]
Type=oneshot
User=root

Environment=STEPPATH=/etc/step-ca \
            CERT_LOCATION=/etc/step/certs/%i.crt \
            KEY_LOCATION=/etc/step/certs/%i.key

; ExecCondition checks if the certificate is ready for renewal,
; based on the exit status of the command.
; (In systemd <242, you can use ExecStartPre= here.)
ExecCondition=/usr/bin/step certificate needs-renewal ${CERT_LOCATION}

; ExecStart renews the certificate, if ExecStartPre was successful.
ExecStart=/usr/bin/step ca renew --force ${CERT_LOCATION} ${KEY_LOCATION}

; Try to reload or restart the systemd service that relies on this cert-renewer
; If the relying service doesn't exist, forge ahead.
; (In systemd <229, use `reload-or-try-restart` instead of `try-reload-or-restart`)
ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active %i.service || systemctl try-reload-or-restart %i"

[Install]
WantedBy=multi-user.target
```

and `cert-renewer@.timer`:

```conf
[Unit]
Description=Timer for certificate renewal of %I
Documentation=https://smallstep.com/docs/step-ca/certificate-authority-server-production
; PartOf=cert-renewer.target

[Timer]
Persistent=true

; Run the timer unit every 5 minutes.
OnCalendar=*:1/5

; Always run the timer on time.
AccuracySec=1us

; Add jitter to prevent a "thundering hurd" of simultaneous certificate renewals.
RandomizedDelaySec=1m

[Install]
WantedBy=timers.target
```

and the necessary Ansible to throw it into place, and synced that over.

Then I created an overrides file for Consul:

```conf
[Service]
; `Environment=` overrides are applied per environment variable. This line does not
; affect any other variables set in the service template.
Environment=CERT_LOCATION="{{ consul.cert_path }}" \
            KEY_LOCATION="{{ consul.key_path }}"
WorkingDirectory="{{ consul.key_path | dirname }}"

; Restart Consul service after certificate renewal
ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active consul.service || systemctl try-reload-or-restart consul.service"
```

Unfortunately, I couldn't build the update the Consul configuration because the TLS certs had expired:

```
TASK [goldentooth.setup_consul : Create a Consul agent policy for each node.] ****************************************************
Wednesday 16 July 2025  18:43:18 -0400 (0:00:57.623)       0:01:24.371 ********
skipping: [bettley]
skipping: [cargyll]
skipping: [dalt]
FAILED - RETRYING: [allyrion -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [harlton -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [erenford -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [inchfield -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [velaryon -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [jast -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [karstark -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [fenn -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [gardener -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [lipps -> bettley]: Create a Consul agent policy for each node. (3 retries left).
FAILED - RETRYING: [allyrion -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [harlton -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [erenford -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [jast -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [inchfield -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [velaryon -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [fenn -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [gardener -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [karstark -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [lipps -> bettley]: Create a Consul agent policy for each node. (2 retries left).
FAILED - RETRYING: [allyrion -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [harlton -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [erenford -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [fenn -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [jast -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [inchfield -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [velaryon -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [gardener -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [lipps -> bettley]: Create a Consul agent policy for each node. (1 retries left).
FAILED - RETRYING: [karstark -> bettley]: Create a Consul agent policy for each node. (1 retries left).
fatal: [allyrion -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [harlton -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [erenford -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [fenn -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [jast -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [inchfield -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [velaryon -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [gardener -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [karstark -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
fatal: [lipps -> bettley]: FAILED! => changed=false
  attempts: 3
  msg: Could not connect to consul agent at bettley:8500, error was <urlopen error [Errno 111] Connection refused>
```

And it was then that I noticed that the dates on all of the Raspberry Pis were off by about 8 days 😑. I'd never set up NTP. A quick Ansible playbook later, every Pi agrees on the same date and time, but now:

```
● consul.service - "HashiCorp Consul"
     Loaded: loaded (/etc/systemd/system/consul.service; enabled; preset: enabled)
     Active: active (running) since Wed 2025-07-16 18:51:09 EDT; 13s ago
       Docs: https://www.consul.io/
   Main PID: 733215 (consul)
      Tasks: 9 (limit: 8737)
     Memory: 19.4M
        CPU: 551ms
     CGroup: /system.slice/consul.service
             └─733215 /usr/bin/consul agent -config-dir=/etc/consul.d

Jul 16 18:51:09 bettley consul[733215]:               gRPC TLS: Verify Incoming: true, Min Version: TLSv1_2
Jul 16 18:51:09 bettley consul[733215]:       Internal RPC TLS: Verify Incoming: true, Verify Outgoing: true (Verify Hostname: true), Min Version: TLSv1_2
Jul 16 18:51:09 bettley consul[733215]: ==> Log data will now stream in as it occurs:
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.903-0400 [WARN]  agent: skipping file /etc/consul.d/consul.env, extension must be .hcl or .json, or config format must be set
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.903-0400 [WARN]  agent: bootstrap_expect > 0: expecting 3 servers
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.963-0400 [WARN]  agent.auto_config: skipping file /etc/consul.d/consul.env, extension must be .hcl or .json, or config format must be set
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.963-0400 [WARN]  agent.auto_config: bootstrap_expect > 0: expecting 3 servers
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.966-0400 [WARN]  agent:  keyring doesn't include key provided with -encrypt, using keyring: keyring=WAN
Jul 16 18:51:09 bettley consul[733215]: 2025-07-16T18:51:09.967-0400 [ERROR] agent: startup error: error="refusing to rejoin cluster because server has been offline for more than the configured server_rejoin_age_max (168h0m0s) - consider wiping your data dir"
Jul 16 18:51:19 bettley consul[733215]: 2025-07-16T18:51:19.968-0400 [ERROR] agent: startup error: error="refusing to rejoin cluster because server has been offline for more than the configured server_rejoin_age_max (168h0m0s) - consider wiping your data dir"
```

It won't rebuild the cluster because it's been offline too long 🙃 So I had to zap a file on the nodes:

```bash
$ goldentooth command bettley,cargyll,dalt 'sudo rm -rf /opt/consul/server_metadata.json*'
dalt | CHANGED | rc=0 >>

bettley | CHANGED | rc=0 >>

cargyll | CHANGED | rc=0 >>

```

and then I was able to restart the cluster.

As it turned out, I had to rotate the Consul certificates anyway, since they were invalid, but I _think_ it's working now.
