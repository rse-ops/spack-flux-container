# The Frankentainer

This build took a lot of manual fixes, basically going back
to the v0.17 release of spack and then several bug fixes:

- `--no-checksum` is ignored in the build / headless mode, so I set it to always True
- The fetcher generates a malformed url
- flux-pmix didn't exist, so I added it, and then had to update all the flux packages
- same with pmix, it needed an update to be compatible

This is my best effort to save the provenance - specifically I pushed my changes
to [this branch](https://github.com/flux-framework/spack/tree/manual-fixes-lammps-efa-jan-2023)
and manually pushed [this container](https://github.com/rse-ops/spack-flux-container/pkgs/container/spack-ubuntu-libfabric/66223339?tag=lammps-zen3) and did a complete automated build for [this one](https://github.com/rse-ops/spack-flux-container/pkgs/container/spack-ubuntu-libfabric/66232607?tag=lammps-zen3-final).

## Instance

The instance we used is `amazon-eks-node-1.23-v20230105` in us-east-2 and instance type is [hpc6a.48xlarge](https://aws.amazon.com/ec2/instance-types/).

To reproduce how I shelled into the instance, I needed to do:

```bash
ssh -o 'IdentitiesOnly yes' -i my-key.pem ec2-user@ec2-xx-xxx-xx-xxx.us-east-2.compute.amazonaws.com
```

Then to prepare docker:

```bash
sudo systemctl start docker
sudo usermod -aG docker $USER
sudo setfacl --modify user:ec2-user:rw /var/run/docker.sock
DOCKER_BUILDKIT=1 docker build --network=host -t <image> .
```

Let's hope these work in production - even a tiny failure of a dependency could
do us in! But I won't cry I'll probably just laugh at this point. Ohh spack.  

Update: we got this working and pushed to [this container](https://github.com/rse-ops/spack-flux-container/pkgs/container/spack-ubuntu-libfabric/69571307?tag=lammps-zen3-working-02-09-2023).

## Local

To build the testing container for `x86_64`, with the same build but for an x86 architecture:

```bash
$ docker build --build-arg spack_cpu_arch=x86_64 -t ghcr.io/rse-ops/spack-ubuntu-libfabric:lammps-x86-64-02-09-2023 .
```

This container was built and pushed to [this image](https://github.com/rse-ops/spack-flux-container/pkgs/container/spack-ubuntu-libfabric/69596129?tag=lammps-x86-64-02-09-2023).
