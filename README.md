# docker-script-find-latest-image-tag

(Over complicated) Bash script to find the version of a Docker image you are running when you took a shortcut and used "latest".  
Now you don't know what image version was previously working, and a breaking change happened..

`docker images` shows `image:<none>` or `image:latest`. What image version am I using?  

Tag your images, and never use latest!

This script checks your local image's ID against every tag of an image in Docker Hub.  

Some issues to note:  
Issue #1: If the image no longer exists on the Docker Registry. You obviously won't find it.  
Issue #2: The script doesn't work for Windows images. There are no manifests available for those.  

## If you're lucky you can find the version tag

### Trick #1

If you're lucky, the image's label will have the version. With a little `jq` magic you can get the good bits.

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
traefik             latest              96c63a7d3e50        2 months ago        85.7MB

$ IMAGE_ID=96c63a7d3e50

$ docker image inspect --format '{{json .}}' "$IMAGE_ID" | jq -r '. | {Id: .Id, Digest: .Digest, RepoDigests: .RepoDigests, Labels: .Config.Labels}'
```

Output:

```bash
{
  "Id": "sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834",
  "Digest": null,
  "RepoDigests": [
    "traefik@sha256:5ec34caf19d114f8f0ed76f9bc3dad6ba8cf6d13a1575c4294b59b77709def39"
  ],
  "Labels": {
    "org.opencontainers.image.description": "A modern reverse-proxy",
    "org.opencontainers.image.documentation": "https://docs.traefik.io",
    "org.opencontainers.image.title": "Traefik",
    "org.opencontainers.image.url": "https://traefik.io",
    "org.opencontainers.image.vendor": "Containous",
    "org.opencontainers.image.version": "v1.7.20"
  }
}
```

### Trick #2

Pull a bunch of images and hope the Image ID matches. Not much to this trick..

If you are unlucky, continue on..

## Requirements

- Docker or Podman
- curl
- jq
- sed
- tr
- grep
- sort
- wc

## Usage

```bash
./docker_image_find_tag.sh -h
Usage:
./docker_image_find_tag.sh [-n image name] [-i image-id]
Example: ./docker_image_find_tag.sh -n traefik -i 96c63a7d3e50 -f 1.7
  -n [text]: Image name (Required). '-n traefik' would reference the traefik image
  -i [text]: Image ID. Required if Image ID (Long) (-L) is ommited.
             Found in 'docker images'. Can also use -i image:tag
  -L [text]: Image ID (Long). Required if Image ID (-i) is ommited
      Example: -L sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
      To find, it is Id from:
      docker image inspect --format '{{json .}}' IMAGE_ID | jq -r '. | {Id: .Id, Digest: .Digest, RepoDigests: .RepoDigests, Labels: .Config.Labels}'
  -D: Use Docker binary for Image ID check (Default) (Optional)
  -P: Use Podman binary for Image ID check (Optional)
  -r [text]: Registry URL to use. Example: -r https://index.docker.io/v2 (Default) (Optional)
  -a [text]: Registry AUTH to use. Example: -a https://auth.docker.io (Default) (Optional)
  -l [number]: Tag limit. Defaults to 100. (Optional)
  -f [text]: Filter tag to contain this value (Optional)
  -v: Verbose output (Optional)
```

## Usage Example

```bash
# ./docker_image_find_tag.sh -n traefik -i 96c63a7d3e50 -f 1.7. -l 10 -v

Using IMAGE_NAME: traefik
Using REGISTRY: https://index.docker.io/v2
Found Image ID Source: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
Found Total Tags: 610
Found Tags (after filtering): 178
Limiting Tags to: 10
Limit reached, consider increasing limit (-l [number]) or use more specific filter (-f [text])

Found Tags:
v1.7.21-alpine
v1.7.21
v1.7.20-alpine
v1.7.20
v1.7.19-alpine
v1.7.19
v1.7.18-alpine
v1.7.18
v1.7.17-alpine
v1.7.17

Checking for image match..
Found match. tag: v1.7.20
Image ID Target: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
Image ID Source: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
```

This example is searching for an image named traefik, with an image ID of `96c63a7d3e50` (which we got from running the command `docker images`.

The total number of tags for traefik images is 610! You could use this script to check every one, but that will take a minute or two. Instead, you can filter the results a bit. I *think* I’m running something in version 1.7.x. I’m also limiting the search to 10 tags in this example to keep the output small.
