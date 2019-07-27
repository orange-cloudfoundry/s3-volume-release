# s3-volume-release

A bosh release to create and mount a s3 bucket based filesystem on cloud foundry applications.

## Installation

Add to your cloud foundry deployment the ops file [add-s3-volume-service.yml](/operations/cf/add-s3-volume-service.yml)

## Usage

You will need a service broker which give [volume_mount](https://github.com/openservicebrokerapi/servicebroker/blob/v2.15/spec.md#volume-mount-object) 
parameter.

It must be in this form:

```json
{
  "volume_mounts": [{
  "driver": "s3driver",
  "container_dir": "/home/vcap/datas3",
  "mode": "rw",
  "device_type": "shared",
  "device": {
    "volume_id": "bc2c1eab-05b9-482d-b0cf-750ee07de311",
    "mount_config": {
      "endpoint": "https://my-s3-server.com",
      "access_key_id": "KEYID",
      "secret_access_key": "ACCESSKEY",
      "bucket": "my-bucket",
      "region": "no-region",
      "mount_options": {}
    }
  }
  }]
}
```

You can pass more config in `mount_config`:

- `endpoint` (*Optional*, string, Default: aws s3): Your s3 endpoint
- `access_key_id` (**Required**, string): S3 access key id
- `secret_access_key` (**Required**, string): S3 access secret
- `bucket` (**Required**, string): The bucket to mount
- `region` (**Required**, string): Region when use aws s3 or supporting s3, you can set any value if your s3 doesn't support it.
- `acl` (*Optional*, string): Acl to use on an s3 compatible
- `subdomain` (*Optional*, bool): Set to true to use domain style for bucket instead of path style.
- `mount_options` (*Optional*, map): Pass mount options to fuse (see manual: http://man7.org/linux/man-pages/man8/mount.fuse.8.html#OPTIONS)
- `storage_class` (*Optional*, string): Storage class to use on an s3 compatible
- `use_content_type` (*Optional*): Set to true to use content type to fetch data
- `use_sse` (*Optional*, boolean): Set to true to use sse (only works on aws s3)
- `use_kms` (*Optional*, boolean): Set to true to use kms (only works on aws s3)
- `kms_key_id` (*Optional*, string): Kms key id (only works on aws s3)
- `region_set` (*Optional*, boolean): set to true for using region set

## Limitation

Under the hood [s3-volume-driver](https://github.com/orange-cloudfoundry/s3-volume-driver) use [goofys](https://github.com/kahing/goofys) 
which as some limitation when using this filesystem:

goofys has been tested under Linux and macOS.

List of non-POSIX behaviors/limitations:
  * only sequential writes supported
  * does not store file mode/owner/group
    * use `--(dir|file)-mode` or `--(uid|gid)` options
  * does not support symlink or hardlink
  * `ctime`, `atime` is always the same as `mtime`
  * cannot `rename` directories with more than 1000 children
  * `unlink` returns success even if file is not present
  * `fsync` is ignored, files are only flushed on `close`

In addition to the items above, the following are supportable but not yet implemented:
  * creating files larger than 1TB

## How it works ?

### Components

there is 6 components engaged:
- `s3driver`: Compliant docker volume driver api 1.12.0 (see: https://docs.docker.com/v17.09/engine/extend/plugins_volume/ )
which can only be called by diego. Cloud foundry diego team make order call different than docker and we use [lib provided by 
cloud foundry persistent](https://github.com/cloudfoundry/volumedriver) team which is giving response content not attended by docker.
- `s3mounter`: Small component which mount volume and be the fuse server compliant for each volume mount by using `your s3 server`.
- `your s3 server`: Your external s3 server
- `Fuse`: to mount filesystem using `s3mounter`
- `diego`: Which made call to `s3driver` for calling mount on local filesystem and give access to this previous volume on app instance.

### Architecture

![architecture](/docs/archi.png)

### Sequence when mounting

![architecture](/docs/sequence.png)

## Why this project ?

1. We wanted our users use s3 instead of filesystem on their app but some of them can't make the change rapidly.
2. We don't wanted to operate something complicated as a storage server and use nfs-volume-service provided by community.
3. We had an s3 server fully operated externally
