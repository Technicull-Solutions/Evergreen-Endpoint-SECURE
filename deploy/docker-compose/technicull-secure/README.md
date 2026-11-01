# Evergreen Endpoint | SECURE on Dockge

This stack is the drop-in replacement for the existing `yopass` deployment at
`https://secure.technicull.ca`. It preserves the externally visible hostname,
the loopback listener on port `1337`, the Compose project and service name used
by the current stack, and membership in `ix-cloudflared_default`.

It does not reuse the current Redis or file-store datasets. The default dataset
root is `/mnt/NVME/evergreen-endpoint-secure`; change `SECURE_DATA_ROOT` in the
Dockge environment if the datasets are created elsewhere.

## Required preparation

1. Confirm that the `cloudflared` external network is named
   `ix-cloudflared_default` on the TrueNAS host.
2. Create the new datasets/directories:
   - `/mnt/NVME/evergreen-endpoint-secure/redis`
   - `/mnt/NVME/evergreen-endpoint-secure/files`
3. Grant the Redis container write access to `redis` and UID `1000` write access
   to `files`. Confirm the actual Redis UID from the pinned image on the host
   before applying dataset ownership; do not make either dataset world-writable.
4. Copy `.env.example` to Dockge's stack environment and pin
   `SECURE_IMAGE_TAG` to a tested release or `sha-<commit>` tag.
5. Pull the image before the change window and confirm the pull succeeds. A
   private GHCR package requires a read-only package credential configured on
   the Docker host; never store that credential in this Compose file.

## Cutover

1. Record the current stack image digest and export its Compose/environment
   configuration for rollback.
2. Stop the existing stack so only one process can bind `127.0.0.1:1337`.
3. Deploy this stack in Dockge and wait for both health checks to pass.
4. From the TrueNAS host, verify `http://127.0.0.1:1337/`, `/health`, and
   `/ready`. The readiness endpoint must return success before public testing.
5. Verify `https://secure.technicull.ca` through the existing reverse proxy:
   load the home page, switch light/dark mode, create a one-time test secret,
   retrieve it once, and confirm a second retrieval fails.
6. Confirm Redis AOF and uploaded-file data are being written only under the new
   dataset root.

## Rollback

Stop this stack, restore the previous Compose/environment configuration and
image digest, and start the previous stack on port `1337`. The new datasets are
isolated and can remain in place for investigation; do not delete either dataset
during the rollback window.
