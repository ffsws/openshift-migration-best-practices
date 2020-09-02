# Best practices for migrating from OpenShift&nbsp;Container&nbsp;Platform&nbsp;3&nbsp;to&nbsp;4

This guide provides recommendations and best practices for migrating from OpenShift Container Platform 3.9+ to OpenShift 4.x with the Migration Tookit for Containers (MTC). 

This section describes how MTC works, when MTC should be used, and alternative tools for migration.

The recommendations are organized into the following sections:

1. **[Planning](./planning.md)**: Considerations and strategies for planning the migration
2. **[Cluster health checks](./cluster-health-checks.md)**: Cluster metrics to check before migration
3. **[Premigration testing](./premigration-testing.md)**: Testing the migration plan before running the actual migration
4. **[Running the migration](./running-the-migration.md)**: Best practices for running the migration
5. **[Troubleshooting](./troubleshooting.md)**: Common troubleshooting tasks

---
## About the Migration Toolkit for Containers

The Migration Toolkit for Containers (MTC) migrates application workloads in two steps:

1. Source cluster is backed up to object storage.
2. Target cluster is restored from object storage.

MTC migrates the following resources and data:

* Kubernetes Resources
  * All namespaced resources are processed, including Custom Resources.  MTC performs a dynamic discovery of all API resources in each referenced namespace.
  * Some cluster scoped resources are processed.  If a namespaced resource references a cluster scoped resource it will be processed.  For example this would include: Persistent Volumes bound to a Persistent Volume Claim, Cluster Role Bindings, and Security Context Constraints
* Persistent Volume data
    * 2 options exist for processing Persistent Volume data
        * Move:  For remote storage (example: NFS) that is accessible by both source and target, it may be possible to only ‘move’ the PV definition from source cluster to target.  This is quickest way to handle PV data migration
        * Copy:  2 methods for 'copying' exist
            * Snapshot:  For storage providers that support snapshots and have a Velero VolumeSnapshotPlugin configured (AWS, Azure, Google) it’s possible snapshot a volume assuming other requirements are met such as same storage/cloud vendor, clusters in same region, etc.
            * Filesystem Copy:  For all other storage use-cases a filesystem copy serves as the generic means of being able to copy data from a source PV to a destination PV.  This is the mechanism to use when changing storage classes in a migration, it grants most flexibility.
* Internal Images
    * Internal images such as the result of `s2i` builds will be processed during a migration.  Each ImageStream reference in a given namespace will be copied to the destination cluster's registry.

## When to use MTC

In an ideal scenario migrating an application from one cluster to another would be a redeploy of the application from a pipeline and perhaps copy of persistent volume data.  For many situations this may not be sufficient as a running application on the cluster may have had adhoc changes done to it over some period of time and it's drifted from the initial deploy.  MTC is designed to handle those cases when a user is not sure exactly what is in a namespace and they want the entire contents migrated to a new cluster.

Our general recommendation is if it's possible to redeploy your application from a pipeline proceed with that option, if not leverage MTC

## Alternative Tools in Upstream

In addition to MTC, there are 2 tools in upstream which may be of interest for larger scale migrations to help only with migration of PVs and Images.
* `pvc-migrate`: https://github.com/konveyor/pvc-migrate
* `imagestream-migrate`: https://github.com/konveyor/imagestream-migrate

The alternative upstream tools are written to be smaller focused tools that are easier to comprehend and debug opposed to Golang based Kubernetes controllers.  The tools leverage a mixture of Ansible + small Python snippets + rsync/skopeo.  

If these tools are used, they would be leveraged by:
1. Run MTC in a configuration that skips processing of PVs and Images
    * Example, in the MigrationController Custom Resource on the host cluster set the below values:
        ```
        disable_image_migration: true
        disable_pv_migration: true
        ```
1. Execute `pvc-migrate` and/or `imagestream-migrate` as per upstream documentation it is expected that MTC would be run in a configuration of skipping processing of pvs/images and the alternative tools would be run

### When to consider using alternative tools vs MTC

 * Large scale migrations
    * ~50+ ImageStreams per Namespace
    * Multiple ~100GB+ Persistent Volumes to be copied

 * Requirements for alternative tools
    * Direction connection available between source and target cluster, i.e. a process on each Node of the source cluster is able to connect to an exposed Route on the target cluster.
    * The host executing pvc-migrate has root ssh access to each Node of the source cluster
    * For pvc-migrate, OCS 3 -> OCS 4 is the only supported path.  No other storage providers are implemented.

 * Considerations of MTC vs alternative tools
    * Serial processing: MTC processes 1 Plan at a time and 1 Backup/Restore operation at a time
        * MTC plans to address this in future to allow parallel execution
            * Related to future contributions in Velero via [velero-#487](https://github.com/vmware-tanzu/velero/issues/487) 
        * Parallel Image migrations are possible using imagestream-migrate
    * Double copy:  Each MTC migration consists of at least 2 copy operations:  Backup copy and Restore copy
        * Alternative tools are a direct single copy from source to target, they don't need to do the 2-step backup/restore process.
    * Ease of debugging/customizing behavior
        * MTC is a collection of golang Kubernetes controllers working together to orchestrate Velero to perform a series of Backup and Restore operations.  Debugging a failure may be challenging, it can span different clusters, namespaces, and controller logs.  In addition, customizing MTC behavior requires golang updates and recompiling code and updating the operator to deliver.
        * Alternative tools leverage Ansible, a small amount of python snippets when essential, and standard tools of rsync and skopeo.  Users are more likely to have a familiarity with Ansible and be able to debug problems and/or customize behavior as they need.
            * An example we've seen in past has been when doing large migrations at scale and running into filesystem corruption issues.  Leveraging `pvc-migrate` with rsync, it is fairly easy to see the rsync error from a corrupted source file and reason through the fix, then address on source volume and re-migrate that PV.  