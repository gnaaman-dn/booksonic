# Local Docker Registry

## Copying an image with a specific manifest-hash

```bash
docker run -it quay.io/skopeo/stable:latest copy \
    --all --dest-tls-verify=false --preserve-digests \
    docker://docker.io/${IMAGE}@sha256:${MANIFEST_HASH} \
    docker://dn19.dev.drivenets.net:5000/${IMAGE}@sha256:${MANIFEST_HASH}
```

```bash
docker run -it quay.io/skopeo/stable:latest copy \
    --all --dest-tls-verify=false --preserve-digests \
    docker://docker.io/debian@sha256:4abf773f2a570e6873259c4e3ba16de6c6268fb571fd46ec80be7c67822823b3 \
    docker://dn19.dev.drivenets.net:5000/debian@sha256:4abf773f2a570e6873259c4e3ba16de6c6268fb571fd46ec80be7c67822823b3
```