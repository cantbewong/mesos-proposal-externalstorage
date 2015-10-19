## What does DVDI Isolator next look like?

1. move mount specification away from environment and into specific configuration like other isolators
2. Allow concurrent use with other isolators (Mesos Containerizer) on same task

---

### Mesos interface changes

The Mesos Containerizer Shared Filesystem isolator currently receives volume related specifications in the ContainerInfo included within ExecutorInfo.

The DVDI isolator currently receives external mount related specifications through environment variables passed through ExecutorInfo.

In order to allow the Mesos Containerizer Shared Filesystem isolator to work with an external mounted volume, it will be necessary to pass a host mount point from the DVDI isolator to the Mesos Containerizer Shared Filesystem isolator at run time. Note that this mount location is not determined until the mount occurs at run time when prepare() is called.

In the interest of consistency, it would be best to move the DVDI external volume specification from environment variables to new ContainerInfo elements.

#### Summary of proposed interface changes:

1. Add new DVDI external mount specification elements to ContainerInfo (and container/volumes within Marathon REST API)
2. Implement a mechanism to allow the DVDI isolator to run before the Mesos Containerizer Shared Filesystem isolator during prepare() AND
3. Allow the DVDI Isolator to feed mount points to the Mesos Containerizer Shared Filesystem isolator
    - TBD: Is it possible to simply overwrite host_path in ContainerInfo/volume or is something more elaborate required?

4. Implement a mechanism for the DVDI isolator to determine the slave work directory.
---

### Isolator interface ordering requirements for concurrent use of DVDI and Mesos Containerizer isolators

Because of interaction related to the external mount, a specific deterministic order will be required to support concurrent use of both isolators.

- On prepare(), *DVDI first*, Mesos Containerizer last
- On cleanup(), *DVDI last*
- On recover(), perhaps either order would work, but choose *DVDI last*, rather than random, on principle that behavior should be deterministic and repeatable

Also need to consider the scenario where an error occurs during execution on the first isolator to process prepare(), cleanup(), and recover().

- Does/Should the second isolator get informed of the error?
- Should the second isolator even be invoked at all?

---

#### General DVDI Isolator internal improvements

- move mount specs from environment to JSON, using google protocol buffer
- if slave's work directory can be made available to the isolator, use it for storing the mount file
- enable use of slave checkpoint code (now in internal namespace)
- add some unit test cases

---

#### Is there an existing Mesos mechanism to get reporting/event publication to allow for central monitoring of mount creation? 

This would be useful for audit and chargeback, and possibly also for centralized troubleshooting.

---
