# Docker images upgrade (v2)

# Why?

Docker are moving to use 'content addressable' image layers - which basically
means they are using a hash (sha256) of the image layer as the image layer id,
before they were using a separate random id. Example image id now looks like:
'sha256:b3549bff6e9847a0170e769bac608be4a6c73a95e628a33a4040dbcd05139fa8'.

It would be advantageous for Triton to also use 'content addressable' images, as
then we don't need a separate IMGAPI.uuid to Docker.id for images, we can just
use the same id (albeit a shortened one for IMGAPI - 32 bytes instead of 64
bytes), and sdc-docker can drop the limitation that an image can only be
referenced in one registry (e.g. currently you cannot re-tag busybox to
quay.io/busybox). Sharing of images should increase, as currently we require
redownloading of the same image when it's from a separate registry, whereas with
a hash of the image layer(s) you can trust it's the same content regardless of
where it came from.

# Docker Terminology

 * *registry v1* - old registry API  (unsupported by docker hub as of Nov 2015)
 * *registry v2* - newest registry API (docker 1.6)
 * *image v1* - old image schema format - uses a random image/layer id
 * *[image v2 schema 1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)* -
   contains sha256 layer info, also contains backward compat v1 info (docker 1.3)
 * *[image v2 schema 2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)* -
   content addressable images, manifest lists (docker 1.10)

# Changes to Triton

All code that uses docker id references will need to be updated to allow and
handle the sha256 prefixed 'sha256:123...' image id format. Mostly this will be
for the image lookup code paths, here are some of the commands that will need
updating:

 * docker build (base image reference, generating sha256 id's, v2 schemas)
 * docker commit (generating sha256 id's, v2 schemas)
 * docker history (image lookup)
 * docker images (image listing)
 * docker inspect (image lookup, v2 schemas)
 * docker pull (image lookup, v2 schemas, imgapi storage)
 * docker push (image lookup, v2 schemas)
 * docker rmi (image lookup)
 * docker tag (image lookup)

## Pull

Require docker images to be pulled in the docker v2 image schema format (either
schema v2.1 or schema v2.2 format). Triton won't support pulling of older v1
images anymore (or we'll have an upgrade schema step to convert these to v2?).

The image `config` (json) is stored in docker_images bucket in Moray, a special
`version` marker will be used to distinguish between different docker image
config formats (in case we need to convert up|down between different schema
versions, or have specific code paths for specific image versions).

## Upgrading Existing Images

Existing images will be left as they are (they will be marked as v1 images),
they will still be valid/used but Triton will need to handle these v1 images
differently, likely in one of these ways:

 * upconvert these old image configs to the v2.1 format (this also requires a
   sha256 generation for existing images)
 * have different code paths for different versions, i.e. for `docker inspect`,
   `history`, etc...)
 * or just report that these v1 images are unsupported - and users need to
   re-download in order to move to the new v2 image schema format

# IMGAPI

For docker images, the IMGAPI image uuid will be the first 32 bytes of the
sha256 of the compressed image (i.e. the raw blob we download from the
registry).

An `uncompressedSha256` will be generated during download (docker pull) and once
fully downloaded will be saved onto the manifest.files[0] object - the
'uncompressedSha256` will be used when pushing images (in the image manifest).

# Docker build/commit

Docker build will need to generate/store the uncompressedSha256 - currently
docker build streams the image data (so it doesn't know the final sha256), so
perhaps an IMGAPI renameUuid step is required so that the image uuid uses the
final compressed sha256 uuid, this rename would occur after upload, but before
the image is activated.








# Outstanding Questions

1. Do we allow pulling older v1 images (how many images/registries only support
   v1 images, note that docker moved to v2)

2. What to do with existing v1 images in sdc-docker (not-supported, upconvert or
   have both)? What happens with billing lookups (i.e. you were running image X)
   if the image is converted (as it will use a different uuid)?




# Appendix (you can ignore from here down, example image schemas and configs)

## Image manifest v2.2

```json
$ H1="Accept: application/vnd.docker.distribution.manifest.v2+json"
$ H2="Accept: application/vnd.docker.distribution.manifest.list.v2+json"
$ curl -k -H "$H1" -H "$H2" -X GET https://192.168.99.100:5000/v2/bbox/manifests/latest | json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 2122,
    "digest": "sha256:b97bf8f04079397cefa8d7a888266c10ee7e8a88813d41c655ead9ae9ca910bc"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar",
      "size": 10240,
      "digest": "sha256:84ff92691f909a05b224e1c56abb4864f01b4f8e3c854e4bb4c7baf1d3f6d652"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar",
      "size": 3573760,
      "digest": "sha256:847f49d214bac745aad71de7e5c4ca2fa729c096ca749151bf9566b7e2e565d9"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar",
      "size": 10240,
      "digest": "sha256:84ff92691f909a05b224e1c56abb4864f01b4f8e3c854e4bb4c7baf1d3f6d652"
    }
  ]
}
```

## Image config "application/vnd.docker.container.image.v1+json" format

Note the 'history' and 'rootfs' properties are now in the config (in v2.1 they
are on the manifest). There is no 'parent' attribute (although there is still a
Config.Image which was the original parent for this image).

```json
$ curl -k -X GET https://192.168.99.100:5000/v2/bbox/blobs/sha256:b97bf8f04079397cefa8d7a888266c10ee7e8a88813d41c655ead9ae9ca910bc | json
{
  "author": "Jérôme Petazzoni <jerome@docker.com>",
  "architecture": "amd64",
  "comment": "",
  "created": "2016-06-08T22:03:43.175Z",
  "config": {
    "AttachStdin": false,
    "AttachStderr": false,
    "AttachStdout": false,
    "Cmd": [
      "/bin/sh"
    ],
    "Domainname": "",
    "Entrypoint": null,
    "Env": null,
    "Hostname": "",
    "Image": "4deca60f7d4ac566e8f88c8af3f89b32c389d49ec4584eb8eafaf5f6cb840c1c",
    "Labels": null,
    "OnBuild": null,
    "OpenStdin": false,
    "StdinOnce": false,
    "Tty": false,
    "User": "",
    "Volumes": null,
    "WorkingDir": ""
  },
  "container_config": {
    "AttachStdin": false,
    "AttachStderr": false,
    "AttachStdout": false,
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) [\"/bin/sh\"]"
    ],
    "Domainname": "",
    "Entrypoint": null,
    "Env": null,
    "Hostname": "",
    "Image": "4deca60f7d4ac566e8f88c8af3f89b32c389d49ec4584eb8eafaf5f6cb840c1c",
    "Labels": null,
    "OnBuild": null,
    "OpenStdin": false,
    "StdinOnce": false,
    "Tty": false,
    "User": "",
    "Volumes": null,
    "WorkingDir": ""
  },
  "history": [
    {
      "created": "2016-06-08T22:03:42.473Z",
      "created_by": "/bin/sh -c #(nop) MAINTAINER Jérôme Petazzoni <jerome@docker.com>",
      "empty_layer": false,
      "author": "Jérôme Petazzoni <jerome@docker.com>"
    },
    {
      "created": "2016-06-08T22:03:43.098Z",
      "created_by": "/bin/sh -c #(nop) ADD file:c36f77eebbf6ed4c99488f33a7051c95ad358df4414dfed5006787f39b3cf518 in /",
      "empty_layer": false,
      "author": "Jérôme Petazzoni <jerome@docker.com>"
    },
    {
      "created": "2016-06-08T22:03:43.175Z",
      "created_by": "/bin/sh -c #(nop) [\"/bin/sh\"]",
      "empty_layer": false,
      "author": "Jérôme Petazzoni <jerome@docker.com>"
    }
  ],
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:84ff92691f909a05b224e1c56abb4864f01b4f8e3c854e4bb4c7baf1d3f6d652",
      "sha256:847f49d214bac745aad71de7e5c4ca2fa729c096ca749151bf9566b7e2e565d9",
      "sha256:84ff92691f909a05b224e1c56abb4864f01b4f8e3c854e4bb4c7baf1d3f6d652"
    ]
  }
}
```

## Image manifest v 2.1

```
{
   "name": "hello-world",
   "tag": "latest",
   "architecture": "amd64",
   "fsLayers": [
      {
         "blobSum": "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      },
      {
         "blobSum": "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      },
      {
         "blobSum": "sha256:cc8567d70002e957612902a8e985ea129d831ebe04057d88fb644857caa45d11"
      },
      {
         "blobSum": "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      }
   ],
   "history": [
      {
         "v1Compatibility": "{\"id\":\"e45a5af57b00862e5ef5782a9925979a02ba2b12dff832fd0991335f4a11e5c5\",\"parent\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"created\":\"2014-12-31T22:57:59.178729048Z\",\"container\":\"27b45f8fb11795b52e9605b686159729b0d9ca92f76d40fb4f05a62e19c46b4f\",\"container_config\":{\"Hostname\":\"8ce6509d66e2\",\"Domainname\":\"\",\"User\":\"\",\"Memory\":0,\"MemorySwap\":0,\"CpuShares\":0,\"Cpuset\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"PortSpecs\":null,\"ExposedPorts\":null,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/bin/sh\",\"-c\",\"#(nop) CMD [/hello]\"],\"Image\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"NetworkDisabled\":false,\"MacAddress\":\"\",\"OnBuild\":[],\"SecurityOpt\":null,\"Labels\":null},\"docker_version\":\"1.4.1\",\"config\":{\"Hostname\":\"8ce6509d66e2\",\"Domainname\":\"\",\"User\":\"\",\"Memory\":0,\"MemorySwap\":0,\"CpuShares\":0,\"Cpuset\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"PortSpecs\":null,\"ExposedPorts\":null,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/hello\"],\"Image\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"NetworkDisabled\":false,\"MacAddress\":\"\",\"OnBuild\":[],\"SecurityOpt\":null,\"Labels\":null},\"architecture\":\"amd64\",\"os\":\"linux\",\"Size\":0}\n"
      },
      {
         "v1Compatibility": "{\"id\":\"e45a5af57b00862e5ef5782a9925979a02ba2b12dff832fd0991335f4a11e5c5\",\"parent\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"created\":\"2014-12-31T22:57:59.178729048Z\",\"container\":\"27b45f8fb11795b52e9605b686159729b0d9ca92f76d40fb4f05a62e19c46b4f\",\"container_config\":{\"Hostname\":\"8ce6509d66e2\",\"Domainname\":\"\",\"User\":\"\",\"Memory\":0,\"MemorySwap\":0,\"CpuShares\":0,\"Cpuset\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"PortSpecs\":null,\"ExposedPorts\":null,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/bin/sh\",\"-c\",\"#(nop) CMD [/hello]\"],\"Image\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"NetworkDisabled\":false,\"MacAddress\":\"\",\"OnBuild\":[],\"SecurityOpt\":null,\"Labels\":null},\"docker_version\":\"1.4.1\",\"config\":{\"Hostname\":\"8ce6509d66e2\",\"Domainname\":\"\",\"User\":\"\",\"Memory\":0,\"MemorySwap\":0,\"CpuShares\":0,\"Cpuset\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"PortSpecs\":null,\"ExposedPorts\":null,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/hello\"],\"Image\":\"31cbccb51277105ba3ae35ce33c22b69c9e3f1002e76e4c736a2e8ebff9d7b5d\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"NetworkDisabled\":false,\"MacAddress\":\"\",\"OnBuild\":[],\"SecurityOpt\":null,\"Labels\":null},\"architecture\":\"amd64\",\"os\":\"linux\",\"Size\":0}\n"
      },
   ],
   "schemaVersion": 1,
   "signatures": [
      {
         "header": {
            "jwk": {
               "crv": "P-256",
               "kid": "OD6I:6DRK:JXEJ:KBM4:255X:NSAA:MUSF:E4VM:ZI6W:CUN2:L4Z6:LSF4",
               "kty": "EC",
               "x": "3gAwX48IQ5oaYQAYSxor6rYYc_6yjuLCjtQ9LUakg4A",
               "y": "t72ge6kIA1XOjqjVoEOiPPAURltJFBMGDSQvEGVB010"
            },
            "alg": "ES256"
         },
         "signature": "XREm0L8WNn27Ga_iE_vRnTxVMhhYY0Zst_FfkKopg6gWSoTOZTuW4rK0fg_IqnKkEKlbD83tD46LKEGi5aIVFg",
         "protected": "eyJmb3JtYXRMZW5ndGgiOjY2MjgsImZvcm1hdFRhaWwiOiJDbjAiLCJ0aW1lIjoiMjAxNS0wNC0wOFQxODo1Mjo1OVoifQ"
      }
   ]
}
```

## Image config v1 format (needed?)

```
{
    "id": "a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9",
    "parent": "c6e3cedcda2e3982a1a6760e178355e8e65f7b80e4e5248743fa3549d284e024",
    "created": "2014-10-13T21:19:18.674353812Z",
    "author": "Alyssa P. Hacker &ltalyspdev@example.com&gt",
    "architecture": "amd64",
    "os": "linux",
    "Size": 271828,
    "config": {
        "User": "alice",
        "Memory": 2048,
        "MemorySwap": 4096,
        "CpuShares": 8,
        "ExposedPorts": {
            "8080/tcp": {}
        },
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "FOO=docker_is_a_really",
            "BAR=great_tool_you_know"
        ],
        "Entrypoint": [
            "/bin/my-app-binary"
        ],
        "Cmd": [
            "--foreground",
            "--config",
            "/etc/my-app.d/default.cfg"
        ],
        "Volumes": {
            "/var/job-result-data": {},
            "/var/log/my-app-logs": {},
        },
        "WorkingDir": "/home/alice",
    }
}
```