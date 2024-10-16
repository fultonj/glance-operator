# FAQ

Here's a curated list of general questions about requirements and design
choices of glance-operator.

## Is it known that glance Pods require PVCs even if the backend is not File?

Even though `File` is not the selected backend, `/var/lib/glance` contains
`os_glance_staging_store` directory used as staging area. When
`import-workflow` or import plugins like `image conversion` or `decompression`
are run, a persistence area is required, and the default storage policy
implemented by the `glance-operator` is to build a `RWO PVC` for each `API`
Pod. However, it is possible to not generate any `PVC` by adding
`storage/external: true` in the Glance CR Spec. By doing this, no PVCs are
provisioned, and the human operator is responsible to provide persistence via
the `extraMounts` interface. The same approach applies to `imageCache`. The
glance-operator configures a `image-cache` directory under the same
`/var/lib/glance` path, and it requires persistence to store cached data.

## I want to deploy Glance with NFS: can the same share be used for both images and staging area?

Yes, even though is technically **not** a recommended solution, it is possible
to define a single `extraMount` and map the same NFS storage to both `images/`
and `os_glance_staging_store/`:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  glance:
    template:
      storage:
        external: true
      extraMounts:
      - extraVol:
        - extraVolType: NFS
          mounts:
          - mountPath: /var/lib/glance
            name: nfs
          volumes:
          - name: nfs
            nfs:
              path: <NFS PATH>
              server: <NFS IP ADDRESS>
```

However, keep in mind that `distributed-image-import` is enabled by default,
and requests will eventually be proxied to the replica that is set as owner of
the requested data (according to `worker_self_reference_url` information). The
above might result in performance concerns when an heavy load is detected on
the same Pod, and while it can be addressed by increasing the number of replicas,
or better, deploying multiple `GlanceAPI` to serve multiple workloads, this is
not currently a supported solution as Glance has no knowledge about any logical
clustering model applied to the deployed architecture.

## I deployed my GlanceAPI with a PVC having a certain size, but conversion requires more space

Our documentation recommends to customize `storage/storageRequest` parameter
to be _at least up to the largest converted image size_. However, there are
situations where this data is not available in advance, and if it is, it might
change during the time.
When a `PVC` is created from the `GlanceAPI` `StatefulSet` template, especially
under the local-storage/LVMS recommendation, it can't be resized, and the only
option is to build a new API, attach it to the same backend with a new PVC or
extraMounts.
This can fit a usual day2 operation scenario, and in the k8s context it can be
summarized by the following steps:

1. Build a new GlanceAPI that is supposed to replace the previous one:

    ```yaml
    spec:
      glance:
        template:
          keystoneEndpoint: api1 # endpoint registered in keystone
          glanceAPIs:
            api1:
              customServiceConfig: <same config present in the api "default">
              storage:
                storageRequest: <NewSize>
            ...
    ```

    a. customize the `storage` interface to request a new PVC (via
    `storage/storageRequest` parameter) or set `external: true` in case it will
    be provided via `extraMounts`.

    b. customize the `customServiceConfig` to inject the same backend available
    for the active API

    c. wait for the new API to be running, then swap the keystone endpoint (via
    the `keystoneEndpoint` parameter in the `Glance` Spec) to register the new
    API and replace the previous one

2. Decommission the previous `GlanceAPI` by patching the `OpenStackControlPlane`

```bash
oc -n openstack patch osctlplane openstack --type=json -p="[{'op': 'remove', 'path': '/spec/glance/template/glanceAPIs/default'}]"
```