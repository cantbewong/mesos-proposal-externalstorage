## new external storage resource

### Resource allocation today:

1. Resource providers advertise what they offer
2. Frameworks are aware of their needs, and subscribe to receive offers
3. Mesos Master acts as "matchmaker" forwarding offers to Frameworks, but Framework ultimately has the final say on accepting an offer

Resource offers and acceptance is done in slave centric tuples (cpu, ram, storage, ...) -- offered by a single slave and consumed through a single slave.

Static and the recent dynamic resource reservations are tied to a specific slave


### Impact of external storage resource support:

#### Slave-centric view of resources needs to change, but it doesn't go completely away

One of the primary attributes of external storage is that is is not tied to a specific slave. However it will often have constraints, and preferred affinities. examples:

- External storage could require pre-installation of a particular driver or software package on a slave.
- may *require* presence of a hardware adapter, e.g. Infiniband
- may *prefer* (or require) that slave has a direct Ethernet connectivity to a particular subnet, or VLAN, e.g. a direct connection to a dedicated storage network.
    - This could interact with network as a resource
- may *prefer* a slave in a particular physical location or region
    - take into account top-of-rack switching constraints
    - take into account failure domains for power, AC, etc - (slave in one failure domain, using storage = half the availability)

#### Some types of external storage call for a *tiered offer and reservation* design

Pooling:
1. Advertise a large pool, e.g. 1,000TB
2. Consumers (framework) accept a fractional amount of the pool, e.g. a 50GB sub-allocation, perhaps with an optional filesystem specification, e.g. ext4.
3. The volume is formatted (TBD, by Framework, or by an Isolator) and retained as a reservation but role/Framework. (TBD: where is this reservation retained)
4. The volume is re-offered for consumption by the framework (TBD: how to implement accept/consume optimization)
5. A framework accepts the offered volume

### Issues to be resolved

#### Where are reservations of external storage retained? Options:

 - retention on a single slave doesn't make sense...
     - Which slave? last one to use it?
         - What if task runs on a different slave next time
         - What if this slave is lost? single point of failure = bad

- Pass this responsibility to external resource Framework? might work, but ...
    - Implementing durable, high availability storage for this could be difficult
    - If Mesos and this external resource architecture is a success, there will be many external resource Frameworks implemented. We want implementation of such a Framework to be easy.
        - Maybe centralized retention of reservations makes sense instead
        - Could the external storage platform itself hold the reservations?
            - Yes for some storage platforms, but no for others
        - If this is done by external resource Frameworks, there should be a reference design with re-usable code

### External storage providers need to advertise far more than simple capacity

### External storage providers will be a *tiered* resource inventory:

- Multiple providers (e.g. Amazon EBS, EMC ScaleIo)
- Within a provider, there can be multiple instances.  (e.g. EBS west region, EBS east region, ScaleIo Aisle 15, ScaleIo Aisle 20)
- Within an instance, there can be multiple pools. (e.g. 2000TB HDD, 500TB SSD)
- For some providers, a pool can be sub-allocated as volumes, which can then be represented as volume resources, likely used in conjunction with persistent reservations.

 ### External storage resources will also advertise constraints, preferences, and attributes

 - constraint examples:
     - slave must have rexray driver installed
     - slave must have Infiniband hardware adapter installed
     - slave must be on VLAN 19
  - preference examples:
      - slave should have a direct connection to

 "inventory" is a tiered
Mesos Mater capabilities in a
"fine grain", open and extensible manner

This means they expose a set of properties that is text tag based and not compiled in to code - Mesos passes the information along to various modules in the usage chain
Information is a combination of hierarchy and key value pairs
Consumers of resource describe needs/desires in a "course grain" fashion.
These should never have fine grain specifications built in, e.g. I want 7200 RPM in position 8 of rack 19 in aisle D.
A "broker" acts as a filter in converting fine grain resource provider capabilities into offers the consumers can understand. This broker is extensible/replaceable
