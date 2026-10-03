# Incident - Slow docker builds and a selinux permission issue

## Date
10-2-2026 9:03PM

## Authors
K., Atul

## Status
Resolved

## Summary
1. Repeated docker builds for the project were slow because of [build cache invalidation](https://docs.docker.com/build/cache/invalidation/). 

2. The docker build failed during the following step. SELinux prevented read access of `pyproject.toml` to the `uv` process during the sync phase. 
```bash
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project
```
## Impact
1. Docker builds are unnecessarily slow resulting in an, avoidable, increase in wait times. 

2. The docker build fails to proceed past the failing stage.

## Root cause
1. The `COPY` stage copied the entire source directory to the image. Any change in the project source invalidates this and all successive stages and they have to be rerun. 

2. The host machine has SELinux set to enforcing mode causing it to refuse read access to `pyproject.toml` file from within the container process. This was verified by inspecting the audit message which is copied below:

> `type=AVC msg=audit(1791050216.198:340): avc:  denied  { read } for  pid=67507 comm="main2" name="pyproject.toml" dev="dm-0" ino=7911163 scontext=system_u:system_r:container_t:s0:c863,c1016 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0

The host file system has the SELinux Context label `unconfined_u:object_r:user_home_t` which cannot be accessed from within the container (which expects label `system_u:system_r:container_t`). 

## Trigger
1. Rebuilding the image to reflect changes to project source.

2. Building the image to run the local development server for testing purposes.

## Resolution
1. The Dockerfile commands were rearranged so that the files `pyproject.toml` and `uv.lock` were mounted (`type=bind`) onto the container and the project dependecies were synced without installing the project itself. The rationale is that the project dependencies change infrequently and installing the project itself should come after in another stage. Subsequently this stage can be satisfied from the build cache until the project dependencies should themselves change. 

2. The project manifest files `pyproject.toml` and `uv.lock` were copied into the image rather than being mounted via a bind mount.

## Detection
1. Error message on the console during the docker build command. Exact command is `docker build -t pyslurm:latest .`

2. AVC denial was identified by a GUI notification by the `SELinux Troubleshoot` applet. 

## Action items
| Action Item                                                                 | Type    | Owner | Bugtracker |
|-----------------------------------------------------------------------------|---------|-------|------------|
| Rewrite the Dockerfile with appropriate layering to utilize build cache     | prevent | Atul  | DONE       |
| Copy the mainfests into the image instead of accessing them via bind mounts | prevent | Atul  | DONE       |

## Lessons Learned
- Docker build stages must be carefully layered to take full advantage of the build cache in order to reduce build times.
- SELinux context labels can cause permission issues when host files are accessed from within the image.
- Adding the mount option of `z` or `Z` depending on the situation can usually resolve such SELinux permission issues.
- `chcon -t TYPE FILE` can be used to temporarily resolve the permission issue but it's not a permanent solution.
- `semanage fcontext` can be used to make persistent changes to the SELinux context of files but repeating this procedure for an arbitrary number of files can prove to be tedious. And it's not a portable solution. 
- Additionally, labeling arbitrary host files with a permissive SELinux context can prove disastrous if a compromised container process gains access to sensitive data.

### What went well

### Where we got lucky

## Timeline
2026-10-03 (all times NPT or UTC +5:45)
- 21:03 I noticed slow docker builds while making incremental additions for a feature I was working on
- 21:30 While investigating the source of the delay, I changed the `COPY` command to utilize a bind mount. This was the first occurence of the AVC denial due to mismatched SELinux context.
- 22:15 Both issues were resolved after changing the Dockerfile configuration and trying the build process again. 
- 23:45 Completed the investigation and made final changes to the Dockerfile. 

## Supporting Information

- https://docs.docker.com/engine/storage/bind-mounts/#configure-the-selinux-label
- https://docs.docker.com/build/cache/invalidation/
- https://docs.docker.com/build/cache/optimize/
- https://docs.astral.sh/uv/guides/integration/docker/#caching
