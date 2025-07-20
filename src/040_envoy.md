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

### Cross-Compilation Investigation

To solve the tcmalloc issue, I attempted to cross-compile Envoy v1.32.0 for ARM64 with `--define tcmalloc=disabled` on Velaryon (the x86 node). This would theoretically produce a Raspberry Pi-compatible binary without the memory alignment problems.

#### Setup Completed
- ✅ Created cross-compilation toolkit with ARM64 toolchain (`aarch64-linux-gnu-gcc`)
- ✅ Built containerized build environment with Bazel 6.5.0 (required by Envoy)
- ✅ Verified ARM64 cross-compilation works for simple C programs
- ✅ Confirmed Envoy source has ARM64 configurations (`//bazel:linux_aarch64`)
- ✅ Found Envoy's CI system officially supports ARM64 builds

#### Fundamental Blocker
All cross-compilation attempts failed with the same error:
```
cc_toolchain_suite '@local_config_cc//:toolchain' does not contain a toolchain for cpu 'aarch64'
```

The root cause is a version compatibility gap:
- Envoy v1.32.0 requires Bazel 6.5.0 for compatibility
- Bazel 6.5.0 predates built-in ARM64 toolchain support
- Envoy's CI likely uses custom Docker images with pre-configured ARM64 toolchains

#### Attempts Made
1. **Custom cross-compilation setup** - Blocked by missing Bazel ARM64 toolchain
2. **Platform-based approach** - Wrong platform type (`config_setting` vs `platform`)
3. **CPU-based configuration** - Same toolchain issue
4. **Official Envoy CI approach** - Same fundamental Bazel limitation

#### Verdict
Cross-compiling Envoy for ARM64 would require either:
- Creating custom Bazel ARM64 toolchain definitions (complex, undocumented)
- Finding Envoy's exact CI Docker environment (may not be public)
- Upgrading to newer Bazel (likely breaks Envoy v1.32.0 compatibility)

**The juice isn't worth the squeeze.** For edge routing on Raspberry Pi, simpler alternatives exist:
- nginx (lightweight, excellent ARM64 support)
- HAProxy (proven load balancer, ARM64 packages available)
- Traefik (modern proxy, native ARM64 builds)
- Caddy (simple reverse proxy, ARM64 support)
