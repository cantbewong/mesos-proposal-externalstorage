# Mesos Management of Persistent External Storage
## Dev Plan
![](https://pbs.twimg.com/profile_images/378800000152512466/d64b637225a3e671799940d5fe13c76b_400x400.png)

---

# Executive Summary

Pursue a Phased approach 
  
1. Start Today:
    - assume an existing external storage platform has been deployed
    - consume volumes from it
2. Tomorrow:
    - compose a storage platform that is provisioned by mesos and runs on Mesos slaves using direct attached storage DAS 

---

Implement an abstraction for

Volume: create/remove/mount/unmount

Goal - abstraction works for:
    - RexRay using external storage
    - Docker Volume API for those workloads that run in a Docker container

---

# Background
## How Mesos works now

---

## Slaves (aka Agents)
- Slaves offer resources (including storage) to the Master
- Slaves advertise their capabilities with a combination of
*resources* + *attributes*. 
- The Master's Allocator module maintains a model of system wide resources.
- Resources and attributes are offered to Frameworks (by the Allocator) as  vectors (tuples). For example resources available on a slave node might include cpu+mem+storage 
 
---
 
## Distinction between Resources and Attributes
- the Allocator tracks consumption of resources by Frameworks, and decrements subsequent resource offers while the resource is in use. 
- Attributes are not considered consumed by a running task, so they are not "usage constrained" or tracked by the Allocator - they are always simply passed though in offers to Frameworks. Attributes are commonly used to guide placement by Marathon.

It might make sense for a read-only volume to be advertised as an attribute, if the storage provider can safely expose this to multiple slaves concurrently. For normal read-write volumes, some facility must track consumption by a task, so resource is the proper advertisement, not attribute

---

## Resource Reservations
### Static Reservation
 A resource can be explicitly earmarked with a role in a slave's configuration. 
   `--resources="cpus(ads):8;mem(ads):4096"`
   
 This has the effect of restricting consumption of the resource to particular framework or usage. 
 
 This is known as a *static reservation*. A static reservation can only be changed by restarting the slave.

---

## Resource Reservations
### Dynamic Reservation
- starts as a resource defined in the slave with a "*" role.
- The slave's resource vector is reported to the Mesos Master, which offers the vector to Frameworks.
- A framework responds to the offer with a "Reserve" response, which includes a role designation in the response

---
 
## Dynamic reservation for Persistent storage
  - The Reserve response does not have to include all the elements of the offer vector.
    - For example the offer could be (cpu+memory+storage), but the Reserve response could be just storage

  Dynamic reservations will mandatory for external persistent storage

---

# Phase 1 proposal

## This is an "straw man" proposal that needs further discussion and evaluation

---

## Slaves must be pre-configured
- Must pre-associate slave with storage provider(s) (e.g. ScaleIo)
  - storage provider (ScaleIo) client software assumed pre-installed on slave
- Slaves report this association to Mesos Master using resources + attributes
- resources and attributes can be specified in /etc/default/mesos-slave, environment variables or command line options

---

## Example of slave *resource* configuration for external storage
`RESOURCES="cpus:2;mem:822;disk:234500;ext-storage-inst1:10000000`

This slave is reporting that it is pre-configured to consume storage from an external storage provider designated as inst1
  - This provider has a max capacity of 10000000

TBD this is a awkward way to report and track capacity, see  slides to follow for more. We need to discuss alternate solutions with Mesosphere

---

## Example of slave *attribute* configuration for external storage
`RESOURCES="cpus:2;mem:822;disk:234500;ext-storage-inst1:10000000"`
`ATTRIBUTES="ext-storage-inst1:scaleio~row8~pool-hdd"`

Note that the attribute "key" duplicates the storage provider instance name       - For the scaleio provider, storage volumes will come from  the **provider-pool** named "pool-hdd" in the **provider-instance** designated "row8"
        - The pool might infer a class of storage such as SSD

---

## Use of ATTRIBUTES to indicate connectable storage "tuples" should work nicely with Marathon
`ATTRIBUTES="scaleio:row8~pool-hdd"`

You can easily constrain an application to run on slave nodes with a specific attribute value using [Marathon Constraints](https://mesosphere.github.io/marathon/docs/constraints.html)

This would allow a task associated with a persistent volume to be restarted on *any* slave node that has an attribute indicating it can connect to the volume's storage pool.

---

## Useful proposed Mesos Feature
[Mesos177](https://issues.apache.org/jira/browse/MESOS-1777) describes a future implementation of a facility that would allow a Framework to dynamically add new slave attributes at task launch time. Assuming we build a ScaleIo Framework in a later phase, this could be used as part of a facility to automate association of slaves to ScaleIo storage pools.

---

## For Discussion: How should external storage capacity be advertised?
Mesos currently expects slaves to provide resources, and thus implements a slave centric model for reporting available resources to the Mesos Master.

External storage does not quite match this slave-centric model.

The "straw man" proposal to report a capacity on each connectable slave has some serious flaws, but best to start with something to make the issues apparent.

---

## Demonstration of the difficulty with reporting a shared pool's capacity at the slave level
Suppose the Mesos operator chooses to allow a total pool of 1,000 TB to be utilized by a Mesos Cluster. 

For simplicity assume 2 slaves have connectivity to this pool. 

Assume a Framework wants to consume (reserve) a 750TB external persistent volume, if it is presented with an offer.

discussion of alternatives based on this scenario...

---

## Alternative #1
### 2 slaves both report the full pool capacity
`RESOURCES="...;ext-storage-inst1:1000000000"`
`RESOURCES="...;ext-storage-inst1:1000000000"`

If each slave advertises the 1,000TB the aggregate is inflated by 2x. Although actual offer vectors (cpu+mem+storage) will not report an impossible to achieve tuple. 

After the Framework reserves 750 TB, vectors from *both* slaves need to be reduced by 750 TB. This will require a change to Mesos?

---

## Alternative #2
### Only 1 slave reports the full pool capacity, others report nothing
`RESOURCES="...;ext-storage-inst1:1000000000"`

If only one slave reports the 1,000 TB, what happens if the slave leaves the cluster - probably nothing good!

---

## Alternative #3
### Each slave reports half the capacity of the pool
`RESOURCES="...;ext-storage-inst1:500000000"`

If both slaves advertise a pro-rata portion, e.g 500 TB in this case, the Framework will not be able to consume a 750 TB volume, even though this should be possible.

If a third slave is added, what happens then? Everybody has to change - this isn't going to work!

---

## For Discussion: Do we even want to advertise a pool capacity, with dynamic volume creation from the pool?
Another alternative is operator (admin) composition of specific volumes in advance, and advertisement of the resulting individual volumes

For example an operator composes a 750GB volume plus a couple 500GB volumes

The slave report a 750GB resource and two 500GB resources

---

## Demonstration of difficulty with reporting remote persistent disk in units of volumes
Assume you compose small, medium and large volumes. Whenever the off-the-shelve volume isn't the perfect size, the task must "round up" resulting in waste

In the example of a 750GB and two 500GB volumes, suppose a Framework requires an 800GB volume and four 100GB volumes. Even though this should fit within the total pool, the result will be two failed requests and a lot of wasted storage on the 3 successful requests

---
 
## Additional issue with reporting capacity in units of volumes

Has similar issue to the MB capacity pool

- Do you report the volume on ALL slaves?, OR
- just one that is arbitrarily chosen? 

---

## Summary of Options
1. Operator (admin) precomposes a collection of volumes. The resource pool consists of this assortment of volumes. OR
2. Operator (admin) designates a pool of external storage. For example 1,000 TB. The resource pool consists of this storage, to be "carved" into volumes at the time a Framework accepts an offer in the form of a RESERVE.

Note: Initial implementation with assume pre-deployed storage, and could even assume pre-provisioned volumes. Maybe just tell mesos what volumes have been pre-provisioned, without implementing an operator API to do volume provisioning


## Attribute Options

Assume storage connectivity constraints are defines as slave attributes

1. Have a user declare constraints in Marathon. OR
2. Have Mesos manage this automatically in the allocation logic, with no Marathon configuration required
---

## Creating a volume

Whether done in advance, or at time of RESERVE, this is required:

1. a "tuple" indicating the source of the storage
    - provider-type
    - provider-instance
    - provider-pool
2. volume size in MB
3. filesystem (e.g. ext4) to be used for formatting
4. a mount point on the slave 
    - It might be better to relieve the operator of this specification by auto- generating a unique mount point 
5. optional: a source volume (this volume will be a snapshot)
6. optional: access mode (RW, RO, default is RW)

---

## Volume lifecycle workflow
- Operator (admin) creates volume
  - The step of formatting the volume will be done on an arbitrary slave which has been associated with the provider
    - We want the format to take place at volume create time because it is time consuming and we want to avoid startup latency when tasks are dispatched. Need to determine how and where we can trigger format code.

---

##TBD: Where to perform volume format.  Options:
1. Imbedded in isolator?
    - don't think isolator is invoked at Reserve time so this may not be feasible
2. Dispatch a Task to do this using Marathon
    - If we do this during a Reserve, what are expectations for timeliness? format can take a long time. Is RESERVE a synchronous operation expecting a rapid response? 
3. Utilize an "out-of-Mesos" host to perform formatting

---

### RexRay CLI will be utilized
new-volume

Function | description | comments
--- | --- | ---
new-volume | Create a new volume | *new-snapshot used if this is a clone*
attach-volume | Attach volume to slave instance |
format-device | Format the attached device |

rexray new-volume --size=10 --volumename=persistent-data
rexray attach-volume --volumeid="vol-425e94af"

Assume volume will be attached only during format, and while an associated  task runs. Thus RexRay detach will also be utilized

---

## Persistent Storage Reservation facility in Mesos 0.23.0+
 A ReservedRole (string) and a DiskInfo have been added to Resource.

 TaskInfo links to Resources

 DiskInfo incorporate a slave mount path, a container mount path and a RW/RO flag. DiskInfo looks like a suitable place to record remote persistent volume metadata. I believe there is already a Mesos ID here, and that this is retained persistently by Mesos.

 The Mesos ID might be sufficient if we maintain a "database" that maps the ID to storage-provider, provider instance and volume-id. 
 
 ---
 
## Proposed Mesos extension
 We'd like to avoid maintaining an external storage provider database. I propose adding these to the Mesos DiskInfo:
 
    - external provider type
    - external provider instance ID
    - external provider volume ID
    - (optional) external provider pool ID. This is optional because providers can likely deduce this via API, if instance and volume IDs are available
    - (optional) Do we want to record a file system type?, or can this be deduced by examination

---

## Reference for proposed extension:
 [see DiskInfo](https://github.com/apache/mesos/blob/09e4ba2faf8eb8f4b82b640b641ec4199a11001d/include/mesos/v1/mesos.proto)

---

# Project Steps
## Open Issues - impacts design and should be resolved ASAP
Determine Resource advertisement philosophy:

1. advertise capacity pool and form volumes at RESERVE time OR
2. form volumes in advance and advertise these volumes

---

# Phase 2+ considerations that should influence architecture
Treating resource management as only a capacity issue is short sighted

    - External storage consumes network bandwidth - even if slave has a dedicated storage network, bandwidth is a finite resource
    - External storage has an finite IOPS per pool

If Mesos really intends to claim to *manage* external persistent storage, taking on bandwidth and IOPs is mandatory, though it might be deferred to a later release.

---

## To be discussed: Could limiting mounts per slave be a "poor man's" replacement for limiting bandwidth and IOPs?
- Instead of managing bandwidth and IOPs, we could mange connections (mounts) per slave
- OR should be do this *in addition to* storage bandwidth and IOPs

---

## Storage bandwidth as a managed resource
Bandwidth is associated with a specific slave, so this is *not* tied to a persistent storage volume reservation

thus slaves should advertise storage bandwidth, and task dispatch should consume it

Units? If goal is to simply achieve balanced task distribution (i.e. avoid placements that crowd external storage using tasks onto the same slave), arbitrary units would be OK

---

## IOPs as a managed resource
### Storage IOPs are associated with a provider's storage pool, and not with a slave.

Thus this is best addressed by tying it directly into a persistent storage reservation.

If we go the direction of managing external persistent storage as a MB pool, rather than a collection of pre-composed volumes, IOPs can also be a pool, consumed at RESERVE time by a Framework.

---

## Example of IOPs as a resource
1. Assume pool-A offers 800TB and 400IOPS
2. Assume pool-B offers 800TB and 1000IOPS
3. Framework X wants a 500TB volume with 900IOPS
Result should be:
  - Decline pool-A offer
  - Accept and RESERVE from the pool-B offer, consuming 500TB and 900IOPS

If we go the way of pre-composed volumes, an IOP evaluation (placement decision) should take place during the composition process.

---

## Categorizing effort into implementation stages

1. Get Mesos + Docker (volume drivers) + marathon working
2. Get Mesos + external storage + any framework working (this is like #1 for Frameworks that don't use Docker containers)  
3. Support Mesos managed storage profiles - Do not assume pre-configured storage volumes, have Mesos manage volume creation on appropriate storage 
4. Mesos framework for storage platform lifecycle management - Distributed software defined storage platforms, like ScaleIo, are deployed by Mesos on slave nodes.
5. Mesos *external* storage + Docker (volume drivers)

---

## Deliverables
1. binary that allocates a volume and formats it
    - runs at volume definition time, or RESERVE time (TBD which time)
    - tentative plan: use RexRay CLI
2. binary that mounts a pre-existing volume on a slave for use by a containerized task
    - tentative assumption: this will run in a Mesos isolator at task start time.
    - plan: call out to RexRay CLI
 
 ---
 
# Appendix
 kubernetes has handling for persistent volumes, including block devices. I may be worthwhile to examine this for ideas.
 [https://github.com/kubernetes/kubernetes/blob/master/docs/design/persistent-storage.md]()
 Kubernetes target use case is management of:
 
     - GCE persistent disks
     - NFS shares
     - AWS EBS stores

The current facility assumes the storage is pre-provisioned, outside kubernetes.
