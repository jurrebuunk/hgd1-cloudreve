# hgd1-cloudreve

[Cloudreve](https://github.com/cloudreve/cloudreve) at **https://files.buunk.org**
on the hgd1 k3s cluster. Flux deploys `deploy/` from this repo; the platform
([hgd1-k8s](https://github.com/jurrebuunk/hgd1-k8s)) points Flux here
(`apps/cloudreve.yaml`) and owns the public routes (`infrastructure/cloudflared/config.yaml`).

| Part | What |
| --- | --- |
| `cloudreve.yaml` | Cloudreve 4.19.1, one pod; settings and users in its database |
| `database.yaml` | Postgres `cloudreve-db`, 2 servers, backed up to S3 (`components/postgres` from hgd1-k8s) |
| `s3-gateway.yaml` | https://files-s3.buunk.org → the bucket `cloudreve` only, for the signed links browsers upload and download with |
| `s3-key.sops.yaml` | The bucket's S3 key, entered once in Cloudreve's admin panel |

Files are in the S3 bucket `cloudreve` (512 GB quota, `labctl s3 bucket show cloudreve`).

**Why the gateway:** Cloudreve's uploader sends `Content-Type:
application/octet-stream` with every chunk without signing it, and Ceph 20
refuses unsigned Content-Type on signed-URL uploads. The gateway drops it
(unless signed), passes only the bucket `cloudreve`, and keeps the Host
header the links are signed for.

Secrets are sops files (`*.sops.yaml`), for the cluster's age key and mine:
`SOPS_AGE_KEY_CMD="ssh-to-age -private-key -i ~/.ssh/id_ed25519" sops <file>`.

To build locally: `cp -r ../hgd1-k8s/components . && kustomize build deploy`
(`components/` is git-ignored; Flux adds it from hgd1-k8s).
