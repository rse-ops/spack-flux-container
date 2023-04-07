# Spack Rocky Linux

Here is a small tutorial for interacting with Flux in a container with Rocky Linux!
First, pull the container:

```bash
$ docker pull ghcr.io/rse-ops/spack-rocky-libfabric:tag-8
```

Shell inside.

```bash
$ docker run -it ghcr.io/rse-ops/spack-rocky-libfabric:tag-8 bash
```

The entrypoint sources spack at `/opt/spack-environment` so you don't need to worry
about that. Flux should be on the path, so we can start a test instance:

```bash
$ flux start --test-size=4
```

And then try submitting a job!

```bash
$ flux mini run hostname
```

To see the versions of dependencies we have installed, spack to the rescue!
Check out the spack lockfile (in the concrete specs section toward the top):

```bash
$ cat /opt/spack-environment/spack.lock | less
```

Here is a quick summary of versions (on Rocky Linux 8):

```
openmpi: 4.1.2
gcc: 8.5.0
libfabric: 1.14.0
flux-core: 0.44.0 
flux-pmix: 0.2.0
flux-sched 0.25.0
```
