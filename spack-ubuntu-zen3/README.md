# Spack Ubuntu zen3

This is intended to be built (manually) on a zen3 system.

## Preparing the instance

This is for an hpc* flavor instance (with centos) on EC2 (us-east-2)

```bash
sudo yum update
sudo yum install -y docker
sudo usermod -a -G docker ec2-user
newgrp docker
sudo systemctl enable docker.service
sudo systemctl start docker.service
sudo setfacl --modify user:ec2-user:rw /var/run/docker.sock

```bash
DOCKER_BUILDKIT=1 docker build --network=host -t ghcr.io/rse-ops/spack-ubuntu-zen3:latest .
```

