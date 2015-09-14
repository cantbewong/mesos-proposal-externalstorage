# Mesos Demo Plan

A full implementation of Mesos managed external storage will require some longer term foundation work to enable cluster wide resources.

In the interest of moving forward with something less than perfect, but still useful, this is a plan for a short term proof of concept demonstration project, that will allow Mesos to manage mounts of pre-existing external volumes.

## Conclusions regarding some longer term foundation development investments

An anonymous module running a master node(s) was deemed a better solution that implementing an external storage management framework

Using "hooks" was recommended to implement failure condition detection - however a "slave lost" hook is not currently implemented

## Conclusions regarding implementation of a short term demonstration

Phase 1 plan will be to use a label (attached to tasks) to declare a storage volume name. The storage volume will be presumed to pre-exist, for example it will be the ID of an existing AWS EBS volume.

Since the volume is pre-configured, the Phase 1 plan will not need an anonymous module and will focus on an isolator.

An ETL workload was suggested for the demo.

1. Extract from a store/database
2. Transform,
3. Load result into target

# Demonstration workflow

1. Compose and format an EBS volume.
2. configure every slave to offer volume
    - Custom resource: ebsvolumes = { vol-7bca078f }
2. Implement Isolator
    - isolator will need to be passed the volume ID
         - labels were suggsted, these can be attached to a task in Marathon
    - mount point in host (slave) can be based on volume name (/foo/rexray/ebsvolumes/vol-7bca078f)
3. For Demo Rexray and new isolator will be presumed pre-installed on slave.
    - There will be a RexRay instance (service/daemon) per storage type - for example if EBS and ScaleIo are both supported volume types, two RexRay daemons will be installed and running on the slave.


# Isolator details

Isolator is a C++ class. It could call out to code in other languages.

See mesos/src/slave/containerizer/isolator.hpp for interface definition

## Isolator functions

1. prepare
2. update
3. recover
4. destroy
5. cleanup

## Docker (and RexRay) Volume Driver API
The demonstration will entail having the isolator call out to a Docker or RexRay Volume API (both of tehse have a silmilar interface). The goal is to demonstrate an ETL workflow using Docker containerized applications.

Volume API functions to be utilized:

1. Create
2. Remove
3. Mount
4. Unmount
5. Path (query)

# We want to map Mesos Isolator functions to Volume API calls

## Mesos Isolator prepare()

Called *before* task is started
This is where we want to accomplished the mount within the slave
Mount might already be in place, prepare() can/should be idempotent with regard to mount

- Does this imply that reference counting is needed? perhaps if multiple tasks on the same slave can demand the same mount

Prepare parameters (in):

- containerId
- executorInfo
- directory
- user

We will need the Volume ID in this function but it may not be available without changing code outside the isolator.

- If Volume Id is not available, use of an environement variable could be a stopgap to get something working

# Mesos cleanup

This is considered the *normal* path

- If multiple users of the mount are supported, would examine mount ref count here.
- Do "lazy" (asynch) unmount, don't need to forcibly unmount or monitor because recover() function would handle this if needed.

# Mesos recover

Cleans up any *unknown* orphaned mounts.

- *known* mounts should be destroyed by the cleanup path, so these don't need to be address here.
