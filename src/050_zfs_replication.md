# ZFS and Replication

So remember back in chapters [28](028_nfs_exports.md) and [31](031_nfs_mounts.md) when I set up NFS exports using a USB thumbdrive? Obviously my crowning achievement as an infrastructure engineer.

After living with that setup for a bit, I finally got my hands on some SSDs. Not new ones, mind you – these are various drives I've accumulated over the years. Eight of them, to be precise:
- 3x 120GB SSDs
- 3x ~450GB SSDs
- 2x 1TB SSDs

Time to do something more serious with storage.

## The Storage Strategy

I spent way too much time researching distributed storage options. GlusterFS? Apparently dead. Lustre? Way overkill for a Pi cluster, and the complexity-to-benefit ratio is terrible. BeeGFS? Same story.

So I decided to split the drives across three different storage systems:
- **ZFS** for the 3x 120GB drives – rock solid, great snapshot support, and I already know it
- **Ceph** for the 3x 450GB drives – the gold standard for distributed block storage in Kubernetes
- **SeaweedFS** for the 2x 1TB drives – interesting distributed object storage that's simpler than MinIO

Today we're tackling ZFS, because I actually have experience with it and it seemed like the easiest place to start.

## The ZFS Setup

I created a role called `goldentooth.setup_zfs` to handle all of this. The basic idea is to set up ZFS on nodes that have SSDs attached, create datasets for shared storage, and then use Sanoid for snapshot management and Syncoid for replication between nodes.

First, let's install ZFS and configure it for the Pi's limited RAM:

```yaml
- name: 'Install ZFS.'
  ansible.builtin.apt:
    name:
      - 'zfsutils-linux'
      - 'zfs-dkms'
      - 'zfs-zed'
      - 'sanoid'
    state: 'present'
    update_cache: true

- name: 'Configure ZFS Event Daemon.'
  ansible.builtin.lineinfile:
    path: '/etc/zfs/zed.d/zed.rc'
    regexp: '^#?ZED_EMAIL_ADDR='
    line: 'ZED_EMAIL_ADDR="{{ my.email }}"'
  notify: 'Restart ZFS-zed service.'

- name: 'Limit ZFS ARC to 128MB of RAM.'
  ansible.builtin.lineinfile:
    path: '/etc/modprobe.d/zfs.conf'
    line: 'options zfs zfs_arc_max=1073741824'
    create: true
  notify: 'Update initramfs.'
```

That ARC limit is important – by default ZFS will happily eat half your RAM for caching, which is not great when you only have 8GB to start with.

## Creating the Pool

The pool creation is straightforward. I'm not doing anything fancy like RAID-Z because I only have one SSD per node:

```yaml
- name: 'Create ZFS pool.'
  ansible.builtin.command: |
    zpool create {{ zfs.pool.name }} {{ zfs.pool.device }}
  args:
    creates: "/{{ zfs.pool.name }}"
  when: ansible_hostname == 'allyrion'
```

Wait, why `when: ansible_hostname == 'allyrion'`? Well, it turns out I'm only creating the pool on the primary node. The other nodes will receive the data via replication. This is a bit different from a typical ZFS setup where each node would have its own pool, but it makes sense for my use case.

## Sanoid for Snapshots

Sanoid is a fantastic tool for managing ZFS snapshots. It handles creating snapshots on a schedule and pruning old ones according to a retention policy. The configuration is pretty simple:

```conf
# Primary dataset for source snapshots
[{{ zfs.pool.name }}/{{ zfs.datasets[0].name }}]
	use_template = production
	recursive = yes
	autosnap = yes
	autoprune = yes

[template_production]
	frequently = 0
	hourly = 36
	daily = 30
	monthly = 3
	yearly = 0
	autosnap = yes
	autoprune = yes
```

This keeps 36 hourly snapshots, 30 daily snapshots, and 3 monthly snapshots. No yearly snapshots because, let's be honest, this cluster probably won't last that long without me completely rebuilding it.

## Syncoid for Replication

Here's where it gets interesting. Syncoid is Sanoid's companion tool that handles ZFS replication. It's basically a smart wrapper around `zfs send` and `zfs receive` that handles all the complexity of incremental replication.

I set up systemd services and timers to handle the replication:

```conf
[Unit]
Description=Syncoid ZFS replication to %i
After=zfs-import.target
Requires=zfs-import.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/syncoid --no-privilege-elevation {{ zfs.pool.name }}/{{ zfs.datasets[0].name }} root@%i:{{ zfs.pool.name }}/{{ zfs.datasets[0].name }}
StandardOutput=journal
StandardError=journal
```

The `%i` is systemd template magic – it gets replaced with whatever comes after the `@` in the service name. So `syncoid@bramble-01.service` would replicate to `bramble-01`.

The timer runs every 15 minutes:

```conf
[Unit]
Description=Syncoid ZFS replication to %i timer
Requires=syncoid@%i.service

[Timer]
OnCalendar=*:0/15
RandomizedDelaySec=60
Persistent=true
```

## SSH Configuration for Replication

Of course, Syncoid needs to SSH between nodes to do the replication. Initially, I tried to set this up with a separate SSH key for ZFS replication. That turned into such a mess that it actually motivated me to finally implement SSH certificates properly (see the previous chapter).

After setting up SSH certificates, I could simplify the configuration to just reference the certificates:

```yaml
- name: 'Configure SSH config for ZFS replication using certificates.'
  ansible.builtin.blockinfile:
    path: '/root/.ssh/config'
    create: true
    mode: '0600'
    block: |
      # ZFS replication configuration using SSH certificates
      {% for host in groups['zfs'] %}
      {% if host != inventory_hostname %}
      Host {{ host }}
        HostName {{ hostvars[host]['ipv4_address'] }}
        User root
        CertificateFile /etc/step/certs/root_ssh_key-cert.pub
        IdentityFile /etc/step/certs/root_ssh_key
        StrictHostKeyChecking no
        UserKnownHostsFile /dev/null
      {% endif %}
      {% endfor %}
```

Much cleaner! No more key management, just point to the certificates that are already being automatically renewed. Sometimes a little pain is exactly what you need to motivate doing things the right way.

## The Topology

The way I set this up, only the first node in the `zfs` group (allyrion) actually creates datasets and takes snapshots. The other nodes just receive replicated data:

```yaml
- name: 'Enable and start Syncoid timers for replication targets.'
  ansible.builtin.systemd:
    name: "syncoid@{{ item }}.timer"
    enabled: true
    state: 'started'
  loop: "{{ groups['zfs'] | reject('eq', inventory_hostname) | list }}"
  when:
    - groups['zfs'] | length > 1
    - inventory_hostname == groups['zfs'][0]  # Only run on first ZFS node (allyrion)
```

This creates a hub-and-spoke topology where allyrion is the primary and replicates to all other ZFS nodes. It's not the most resilient topology (if allyrion dies, no new snapshots), but it's simple and works for my needs.

## Does It Work?

Let's check using the goldentooth CLI:

```bash
$ goldentooth command allyrion 'zfs list'
allyrion | CHANGED | rc=0 >>
NAME         USED  AVAIL  REFER  MOUNTPOINT
rpool        546K   108G    24K  /rpool
rpool/data    53K   108G    25K  /data
```

Nice! The pool is there. Now let's look at snapshots:

```bash
$ goldentooth command allyrion 'zfs list -t snapshot'
allyrion | CHANGED | rc=0 >>
NAME                                                        USED  AVAIL  REFER  MOUNTPOINT
rpool/data@autosnap_2025-07-18_18:13:17_monthly               0B      -    24K  -
rpool/data@autosnap_2025-07-18_18:13:17_daily                 0B      -    24K  -
rpool/data@autosnap_2025-07-18_18:13:17_hourly                0B      -    24K  -
rpool/data@autosnap_2025-07-18_19:00:03_hourly                0B      -    24K  -
rpool/data@autosnap_2025-07-18_20:00:10_hourly                0B      -    24K  -
...
rpool/data@autosnap_2025-07-19_14:00:15_hourly                0B      -    24K  -
rpool/data@syncoid_allyrion_2025-07-19:10:45:32-GMT-04:00     0B      -    25K  -
```

Excellent! Sanoid is creating snapshots hourly, daily, and monthly. That last snapshot with the "syncoid" prefix shows that replication is happening too.

And on the replica nodes? Let me check which nodes have ZFS:

```bash
$ goldentooth command gardener 'zfs list'
gardener | CHANGED | rc=0 >>
NAME         USED  AVAIL  REFER  MOUNTPOINT
rpool        600K   108G    25K  /rpool
rpool/data    53K   108G    25K  /rpool/data
```

The replica has the same dataset structure. And the snapshots?

```bash
$ goldentooth command gardener 'zfs list -t snapshot | head -5'
gardener | CHANGED | rc=0 >>
NAME                                                        USED  AVAIL  REFER  MOUNTPOINT
rpool/data@autosnap_2025-07-18_18:13:17_monthly               0B      -    24K  -
rpool/data@autosnap_2025-07-18_18:13:17_daily                 0B      -    24K  -
rpool/data@autosnap_2025-07-18_18:13:17_hourly                0B      -    24K  -
rpool/data@autosnap_2025-07-18_19:00:03_hourly                0B      -    24K  -
```

Perfect! The snapshots are being replicated from allyrion to gardener. The replication is working.

## Performance

How's the performance? Well... it's ZFS on a single SSD connected to a Raspberry Pi. It's not going to win any benchmarks:

```bash
$ goldentooth command_root allyrion 'dd if=/dev/zero of=/data/test bs=1M count=100'
allyrion | CHANGED | rc=0 >>
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.205277 s, 511 MB/s
```

511 MB/s writes! That's... actually surprisingly good for a Pi with a SATA SSD over USB3. Clearly the ZFS caching is helping here, but even so, that's plenty fast for shared configuration files, build artifacts, and other cluster data.
