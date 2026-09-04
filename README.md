# Kargo Helm Example

This is a GitOps repository of a Kargo Helm example for getting started.

### Features:

* Multi-tenant setup: 5 tenants (safti-fr, safti-es, safti-de, safti-pt, megagence-fr),
  each with its own `preprod`/`prod` Stage pair (10 Stages total)
* Per-tenant, per-env Warehouses monitoring their own image tag pattern
  (`preprod-<tenant>-*`, `prod-<tenant>-*`) — no image promoted across Stages,
  since each env/tenant combination is built independently by CI (matching a
  build-per-env-per-tenant pipeline, e.g. Symfony `cache:warmup` baked at build time)
* Feature flag promotion still cascades `preprod-<tenant> -> prod-<tenant>`,
  since config genuinely is the same artifact promoted forward
* Promotion opens a Pull Request and waits for it to be merged before proceeding
  (see `kargo/promotiontask.yaml`)

## Requirements

* Kargo v1.3 (for older Kargo versions, switch to the release-X.Y branch)
* GitHub and a container registry (GHCR.io)
* `git` and `docker` installed

## Instructions

1. Fork this repo, then clone it locally (from your fork).
2. Run the `personalize.sh` to customize the manifests to use your GitHub
   username:

   ```shell
   ./personalize.sh <yourgithubusername>
   ```
3. `git commit` the personalized changes:

   ```shell
   git commit -a -m "personalize manifests"
   git push
   ```
4. Create a guestbook container image repository in your GitHub account. 

   The easiest way to create a new ghcr.io image repository, is by retagging and 
   pushing an existing image with your GitHub username:

   ```shell
   docker login ghcr.io

   docker buildx imagetools create \
     ghcr.io/akuity/guestbook:latest \
     -t ghcr.io/<yourgithubusername>/guestbook:v0.0.1
   ```

   You will now have a `guestbook` container image repository. e.g.:
   https://github.com/yourgithubusername/guestbook/pkgs/container/guestbook

5. Change guestbook container image repository to public.

   In the GitHub UI, navigate to the "guestbook" container repository, Package
   settings, and change the visibility of the package to public. This will allow
   Kargo to monitor this repository for new images, without requiring you to 
   configuring Kargo with container image repository credentials.

   ![change-package-visibility](docs/change-package-visibility.png)

6. Download and install the latest CLI from [Kargo Releases](https://github.com/akuity/kargo/releases/latest)

   ```shell
   ./download-cli.sh /usr/local/bin/kargo
   ```

7. Login to Kargo:

   ```shell
   kargo login --admin https://<kargo-url>
   ```

8. Apply the Kargo manifests:

   ```shell
   kargo apply -f ./kargo
   ```

9. Add the Git repository credentials to Kargo. This can also be done in the UI
   in the `kargo-helm` Project.

   ```shell
   kargo create credentials github-creds \
     --project kargo-helm \
     --git \
     --username <yourgithubusername> \
     --repo-url https://github.com/<yourgithubusername>/kargo-helm.git
   ```

   As part of the promotion process, Kargo requires privileges to commit changes
   to your Git repository, as well as the ability to create pull requests. Ensure
   that the given token has these privileges.

10. Promote the image!

    You now have a Kargo Pipeline per tenant, each with a `preprod` and `prod`
    Stage. Visit the `kargo-helm` Project in the Kargo UI to see the 10 Stages
    (5 tenants x 2 envs).

    ![pipeline](docs/pipeline.png)

    Each `preprod-<tenant>` Stage pulls its image directly from its own
    Warehouse (simulating a CI that builds a distinct image per env/tenant).
    To promote, click the target icon to the left of a `preprod-<tenant>`
    Stage, select the detected Freight, and click `Yes` to promote. This opens
    a Pull Request and waits for it to be merged before syncing Argo CD. The
    `features` Freight (feature flags), unlike the image, does cascade forward
    to the corresponding `prod-<tenant>` Stage once promoted.


## Simulating a release

To simulate a release, retag an image matching one tenant/env's tag pattern.
e.g. for `safti-fr` preprod:

```shell
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/<yourgithubusername>/guestbook:preprod-safti-fr-v0.0.2
```

Then refresh the Warehouse in the UI to detect the new Freight.


## Promoting a feature

Edit the `base/feature-flags.yaml` with a new setting. This will be detected
by the `features` Warehouse as a promotable configuration.
