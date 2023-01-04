# Spack Flux Containers

This was a hard set of containers to get right (to have Flux work in entirely with the
flux operator) so I am providing them as a base here. They are intended to be used
as a base for Flux operator images, meaning that the operator adds a curve
certificate, broker.toml, and handles orchestration.

The trick was having a build of flux core, sched, and pmix with pinned versions
(this credit goes to [milroy](https://github.com/milroy) but then ensuring
that spack uses the system Python, where we can easily install the other Flux
dependency set (needed at runtime). For some reason when we install Python alongside
spack, no matter what we do with `PYTHONPATH` or anything else, they are not found.
Since the container requires a root user with a flux user existing, we run
commands as follows:

```bash
$ sudo -u flux -E PYTHONTPATH=$PATHONPATH 
```

and that seems to work, but it requires sourcing the spack environment file and
adding the flux user and view path to the sudoers file. This took me days to figure out,
and that was many days before that trying other ways to build this combination. 
Also note this container requires a flux option to run. 

```bash
$ flux mini run -ompi=openmpi@5 -n 2 ./osu_get_latency
```
So in the flux operator container we would export `FLUX_OPTION_FLAGS=-ompi=openmpi@5`
Aye, yay yay!
