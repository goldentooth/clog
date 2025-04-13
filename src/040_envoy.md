## Envoy

I would like to replace Nginx with an edge routing solution of Envoy + Consul. Consul is setup, so let's get cracking on Envoy.

Unfortunately, it doesn't work out of the box:

```bash
$ envoy --version
external/com_github_google_tcmalloc/tcmalloc/system-alloc.cc:625] MmapAligned() failed - unable to allocate with tag (hint, size, alignment) - is something limiting address placement? 0x17f840000000 1073741824 1073741824 @ 0x5560994c54 0x5560990f40 0x5560990830 0x5560971b6c 0x556098de00 0x556098dbd0 0x5560966e60 0x55608964ec 0x556089314c 0x556095e340 0x7fa24a77fc
external/com_github_google_tcmalloc/tcmalloc/arena.cc:58] FATAL ERROR: Out of memory trying to allocate internal tcmalloc data (bytes, object-size); is something preventing mmap from succeeding (sandbox, VSS limitations)? 131072 632 @ 0x5560994fb8 0x5560971bfc 0x556098de00 0x556098dbd0 0x5560966e60 0x55608964ec 0x556089314c 0x556095e340 0x7fa24a77fc
Aborted
```

That's because of [this issue](https://github.com/envoyproxy/envoy/issues/23339).

I don't really have the horsepower on these Pis to compile Envoy, and I don't want to recompile the kernel, so for the time being I think I'll need to run a special build of Envoy in Docker. Unfortunately, I can't find a version that both 1) runs on Raspberry Pis, and 2) is compatible with a current version of Consul, so I think I'm kinda screwed for the moment.
