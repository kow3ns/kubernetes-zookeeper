# Docker Image
The docker image contained in this repository is comprised of a base Ubuntu 16.04 image using the latest release of the 
OpenJDK JRE based on the 1.8 JVM and the latest stable release of ZooKeeper, 3.4.10. Ubuntu is a much larger image than 
BusyBox or Alpine, but these images contain mucl or ulibc. This requires a custom version of OpenJDK to be built 
against a libc runtime other than glibc. No vendor of the ZooKeeper software supplies or verifies the software against 
such a JVM, and, while Alpine or BusyBox would provide smaller images, we have prioritized a well known environment.

The image is built such that the ZooKeeper process is designated to run as a non-root user. By default, this user is 
zookeeper. The ZooKeeper package is installed into the /opt/zookeeper directory, all configuration is sym linked into 
the /usr/etc/zookeeper/, and all executables are sym linked into /usr/bin. The ZooKeeper data directories are contained 
in /var/lib/zookeeper. This is identical to the RPM distribution that users should be familiar with.

## Makefile 
The [makefile](Makefile) contained in the docker directory has three commands.
- The `build` command will build the Docker image locally.
- The `push` command will push the image, provided you have correct permissions, 
to grc.io/containers repository.
- The `all` command will perform the `build` command.

The [scripts](scripts) directory contains useful utilities for managing the 
ZooKeeper process in a Kubernetes cluster. 