# hgd1-cloudreve

[Cloudreve](https://github.com/cloudreve/cloudreve) at **https://cloudreve.buunk.org**
on the hgd1 k3s cluster. Flux deploys `deploy/` from this repo; the platform
([hgd1-k8s](https://github.com/jurrebuunk/hgd1-k8s)) points Flux here
(`apps/cloudreve.yaml`) and owns the public routes (`infrastructure/cloudflared/config.yaml`).

| Part | What |
| --- | --- |
| `cloudreve.yaml` | Cloudreve 4.19.1 with our patches (`image/`), one pod; settings and users in its database |
| `database.yaml` | Postgres `cloudreve-db`, 2 servers, backed up to S3 (`components/postgres` from hgd1-k8s) |
| `s3-gateway.yaml` | https://files-s3.buunk.org → the bucket `cloudreve` only, for the signed links browsers upload and download with |
| `s3-key.sops.yaml` | The bucket's S3 key, entered once in Cloudreve's admin panel |
| `oidc.sops.yaml` | Secret of the Keycloak client `cloudreve` |

Files are in the S3 bucket `cloudreve` (512 GB quota, `labctl s3 bucket show cloudreve`).

**Why the gateway:** Cloudreve's uploader sends `Content-Type:
application/octet-stream` with every chunk without signing it, and Ceph 20
refuses unsigned Content-Type on signed-URL uploads. The gateway drops it
(unless signed), passes only the bucket `cloudreve`, and keeps the Host
header the links are signed for.

## Network

- HTTPS only: `http://` redirects and HSTS is set, by the `buunk.org` zone
  in Cloudflare (hgd1-k8s, Public access).
- Pods reach the cluster, the internet and S3 (`s3.lan.buunk.org:443`), not
  the rest of the site: `components/egress/default` and `s3` from hgd1-k8s
  in `deploy/kustomization.yaml`; the database gets its own policy from
  `components/postgres`.

## Login through Keycloak

People sign in with **Sign in with buunk** (Keycloak, realm `hgd1`). OIDC
login is a Pro feature upstream, so `image/` adds it as a patch: no fork,
only the patch files in this repo.

- **Who gets in:** the client roles of `cloudreve` in Keycloak, set per
  group in hgd1-k8s (`platform/keycloak-realm/hgd1.yaml`): `admins` →
  `cloudreve-admin` → Cloudreve group Admin; `family`, `friends` →
  `cloudreve-user` → User. `cloudreve-admin` makes someone an admin in
  Cloudreve only, not in Keycloak.
  No role, no access. The group is set again at every sign-in, so moving
  someone between Keycloak groups changes their rights next time they sign in.
- **Accounts:** the first sign-in creates the Cloudreve account and links it
  to the Keycloak user (its `sub`, in the account's settings). An existing
  Cloudreve account with the same email is linked only when the email is
  **verified** in Keycloak (tick *Email verified* on the user); otherwise
  the person sees "an administrator has to mark your email address as
  verified". An account linked to one Keycloak user can't be taken by
  another.
- **Profile picture:** the picture people set in the hub (buunk-hub) is
  their Keycloak `picture` attribute, and comes along as the `picture`
  claim. At every sign-in it becomes the account's avatar (only an
  `https://` address); Cloudreve's avatar URL then redirects to it. No file
  is kept, so it survives the pod restarting (`data` is an emptyDir). Signing
  in without one clears an avatar set that way; an avatar uploaded in
  Cloudreve itself stays until a hub picture replaces it. A new picture
  shows after the next sign-in here.
- **Password login stays** for accounts that have a password: keep one
  local admin as a way in when Keycloak is down. Accounts made by Keycloak
  have no password.
- **Removing someone:** take them out of the groups in Keycloak *and*
  disable the account in Cloudreve (admin panel → Users). Cloudreve's own
  session lasts until its refresh token runs out, which Keycloak can't end.
- Logging out of Cloudreve doesn't log out of Keycloak.

Settings are the `OIDC_*` variables in `cloudreve.yaml`; the client secret
is `oidc.sops.yaml`, the same value as `client-cloudreve.sops.yaml` in
hgd1-k8s.

## The image

`image/` builds `ghcr.io/jurrebuunk/hgd1-cloudreve`: upstream's source at
pinned commits, our patches applied (`patches/backend`, `patches/frontend`),
built, and only the binary swapped into upstream's own image. The build runs
the patch's tests. CI (`.github/workflows/image.yml`) publishes
`<image/VERSION>` from `main` and never overwrites a version.

| Patch | What |
| --- | --- |
| backend | `service/oidclogin` (flow, ID token checks, account linking, picture as avatar, tests), `routers/controllers/oidc_login.go`, four routes under `/api/v4/session/oidc`, one field `oidc_sub` in user settings, and the avatar endpoint redirecting to a picture address (`service/user/setting.go`) |
| frontend | The button on the sign-in page and the page `/session/oidc` that finishes the sign-in |

How a sign-in goes: `/api/v4/session/oidc/login` sends the browser to
Keycloak (code flow with PKCE, `state` and `nonce` in an encrypted HttpOnly
cookie); `…/callback` checks them, redeems the code with the client secret,
verifies the ID token and leaves a single-use handle (60 s) in a second
HttpOnly cookie; the page `/session/oidc` trades it at `…/token` for a normal
Cloudreve session. No token or code ends up in a URL the app sees.

**Changing a patch:** clone upstream at the commits in `image/Dockerfile`,
apply the patches, change and test, then export again:

```sh
git clone https://github.com/cloudreve/cloudreve && cd cloudreve
git checkout <CLOUDREVE_COMMIT> && git apply ../hgd1-cloudreve/image/patches/backend/*.patch
# … change, go test ./service/oidclogin/ (OIDC_TEST_PG=<dsn> for Postgres) …
git add -A && git commit -m "OpenID Connect login" && git format-patch -1 --zero-commit --no-signature -o ../hgd1-cloudreve/image/patches/backend/
```

Then bump `image/VERSION` (`4.19.1-oidc.N+1`), push, and put the new tag and
digest (from the workflow's summary) in `deploy/cloudreve.yaml`.

**Upgrading Cloudreve:** set `CLOUDREVE_VERSION`, `CLOUDREVE_COMMIT` (the
tag's commit) and `FRONTEND_COMMIT` (its `assets` submodule:
`git ls-tree <tag> assets`) and the final `FROM` (tag and digest) in
`image/Dockerfile`, set `image/VERSION` to `<version>-oidc.1`. A patch that
no longer applies fails the build; fix it as above.

Secrets are sops files (`*.sops.yaml`), for the cluster's age key and mine:
`SOPS_AGE_KEY_CMD="ssh-to-age -private-key -i ~/.ssh/id_ed25519" sops <file>`.

To build locally: `cp -r ../hgd1-k8s/components . && kustomize build deploy`
(`components/` is git-ignored; Flux adds it from hgd1-k8s).
