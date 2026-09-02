# Changes made to run nf-core/rnaseq on AWS HealthOmics

Active account: `677276073900` (external, BYO-ECR mirror), region: `us-east-1`.
Workbench workspace: `bugbash-nextflowonaws` (AWS account `535002863113`).
Branch: `aws-ecr-cache`.

An earlier iteration of this same mirror setup (pull-through-cache + static Wave
mirror) was built directly in the `535002863113` account for the
`daniel_exploration_aws` workspace. `677276073900` is a from-scratch rebuild of
that pattern in a separate, external AWS account, to prove out a "bring your own
ECR mirror" model — a workspace/account owns its own mirror rather than sharing
one baked into the platform account. `nextflow.config` now points at whichever
account is set in `ecrRegistry` (currently `677276073900`); repointing at another
mirror account only requires changing that one value.

## 1. Repo changes

### `nextflow.config`
- Introduced `def ecrRegistry = '677276073900.dkr.ecr.us-east-1.amazonaws.com'`
  as the single source of truth for which account/region hosts the mirror.
- `docker.registry`: `quay.io` → `"${ecrRegistry}/quay"`. Only rewrites *bare*
  image names (e.g. `biocontainers/fastqc:...`) that carry no registry host —
  covers every `biocontainers/*` and `nf-core/ubuntu` container in the pipeline
  via ECR's `quay` pull-through-cache prefix.
- Added a `process { withName: ... }` block with ~28 selectors overriding container
  images for every Seqera Wave container (`community.wave.seqera.io/library/*`).
  These carry a fully-qualified host already, so `docker.registry` can't rewrite
  them — each had to be pointed explicitly at its mirrored ECR copy under
  `def waveRegistry = "${ecrRegistry}/community/library"`, so each selector only
  supplies `${waveRegistry}/image:tag`. Selectors use `(.*:)?PROC(_.*)?` so
  aliased processes importing the same module (e.g. `STAR_ALIGN_IGENOMES`) are
  covered.
- Not yet mirrored: `nvcr.io/nvidia/clara/clara-parabricks:4.6.0-1`
  (`modules/nf-core/parabricks/rnafq2bam/main.nf`), a third upstream registry.
  Currently a dead path — only reachable via the opt-in `use_parabricks_star` GPU
  acceleration flag, which `params.yaml` doesn't set — but it will fail the same
  "repository not found" way quay/Wave images did before mirroring if anyone
  flips that flag on this account.

### `params.yaml` (new file, repo root)
Used as the parameter set passed to HealthOmics `StartRun`. Points at a small
test dataset (mirrors `conf/test.config`, not `conf/test_full.config`) staged
entirely in S3 so no internet egress is needed:
- `input`, `fasta`, `gtf`, `gff`, `transcript_fasta`, `additional_fasta`,
  `bbsplit_fasta_list`, `hisat2_index`, `salmon_index`, `kraken_db` → all point at
  `s3://v0-saas-dev-us-east-1-workbench/nextflow_resources-bugbash-nextflowonaws/rnaseq_test_data/`
- `outdir: '/mnt/workflow/pubdir'` — **the key HealthOmics-specific fix.** Must be
  a local container path, not a raw S3 URI. HealthOmics stages this directory out
  to the run's `outputUri` automatically after the run finishes; Nextflow itself
  never writes to S3 mid-run. Using a raw `s3://...` URI here caused
  `NoSuchKeyException` crashes (see §3) because HealthOmics's own S3 filesystem
  provider mishandles `Files.isDirectory()` checks against not-yet-existing S3
  keys — every mid-run write to a brand-new S3 prefix (pipeline_info params dump,
  software_versions.yml collation, etc.) hit this bug.

### Local test data staged and uploaded
Since the pipeline's built-in `test` profile pulls FASTQs/refs from
`raw.githubusercontent.com`, and this HealthOmics run has no internet egress,
all of it was re-hosted in S3:
- 11 FASTQs (`GSE110004` test reads)
- 8 reference files (`genome.fasta`, `genes_with_empty_tid.gtf.gz`, `genes.gff.gz`,
  `transcriptome.fasta`, `gfp.fa.gz`, `bbsplit_fasta_list.txt`, `hisat2.tar.gz`,
  `salmon.tar.gz`, `kraken2.tar.gz`)
- 2 extra fastas referenced *inside* `bbsplit_fasta_list.txt`
  (`GCA_009858895.3_ASM985889v3_genomic.200409.fna`, `chr22_23800000-23980000.fa`)
- Rewritten `samplesheet_test_s3.csv` and `bbsplit_fasta_list_s3.txt` pointing at
  the S3 copies instead of GitHub URLs. These two files embed their own `s3://`
  URLs internally (not just their own location), so moving buckets/prefixes means
  re-uploading them too, not just re-pointing `params.yaml`.

All uploaded to
`s3://v0-saas-dev-us-east-1-workbench/nextflow_resources-bugbash-nextflowonaws/rnaseq_test_data/`
— this is `bugbash-nextflowonaws`'s own Workbench-managed S3 resource
(`nextflow_resources`), not a shared/general-purpose prefix. Workspace S3
resources appear to be IAM-scoped per workspace by prefix (matches the
`<resource>-<workspace-id>/` naming convention), so data staged under one
workspace's prefix likely isn't readable from another workspace's run role.

## 2. AWS ECR setup

The pipeline's containers come from two upstream registries; each needed a
different fix. Set up twice: once in `535002863113` (original), once from
scratch in `677276073900` (this account, described below) to validate the BYO
pattern.

### `quay.io` (`biocontainers/*`, `nf-core/ubuntu`) — ECR pull-through cache
- Pull-through-cache rule: `aws ecr create-pull-through-cache-rule
  --ecr-repository-prefix quay --upstream-registry quay --upstream-registry-url quay.io`
- **Registry policy** (`aws ecr put-registry-policy`) granting `omics.amazonaws.com`
  `ecr:CreateRepository` + `ecr:BatchImportUpstreamImage` on `repository/quay/*`.
  This is what lets HealthOmics trigger a *fresh* pull-through fetch (create repo +
  import from upstream) the first time it needs an image. Without this, HealthOmics
  can only check "does this already exist?" and fails with "repository not found"
  on anything never previously pulled — it can't trigger the fetch itself.
- **Repository-creation template** (`aws ecr create-repository-creation-template`,
  later `update-repository-creation-template` — `--prefix quay --applied-for
  PULL_THROUGH_CACHE`) so every repo ECR auto-creates under this prefix going
  forward automatically gets the repository policy below, without needing to be
  patched by hand each time. A manually `create-repository`'d repo under the
  `quay/` prefix does **not** get linked to the pull-through-cache rule or this
  template — only a repo ECR auto-creates *during an actual pull-through fetch*
  is linked to the upstream. Confirmed by testing (probe repo created, found
  disconnected, deleted).
- **Repository policy** baked into the template, applied to every auto-created
  `quay/*` repo, with two statements:
  - `omics.amazonaws.com` service principal: `ecr:BatchGetImage` +
    `ecr:GetDownloadUrlForLayer` + `ecr:BatchCheckLayerAvailability`. Lets
    HealthOmics actually *read* an image once it exists — separate from the
    registry policy above, which only covers creating/fetching it.
  - Cross-account trust for the Workbench pod account (`535002863113`, from
    `wb pod describe --pod=aws --org=verily1`), same three read actions, gated on
    `aws:PrincipalTag/vwb-<workspace-uuid>` being `reader`/`writer` (workspace
    UUID from `wb workspace list --format=JSON`). Missed on the first pass —
    documented at
    https://support.workbench.verily.com/docs/guides/workspaces/aws_external_resources/
    — without it, Workbench's own cross-account execution role (not just the
    HealthOmics service principal) can't read the mirror.
- In `535002863113` (original setup): manually warmed 5 images that
  pull-through-cache had never fetched before (`bioconductor-tximeta`, `python`
  — 3 tags, `stringtie`, `subread`, `sylph`) via local `docker pull` through the
  ECR endpoint, as a stopgap before the registry policy above was in place. In
  `677276073900`, only 1 image (`biocontainers/fastqc`) has been warmed so far as
  a smoke test of the rebuilt policy chain — the rest lazy-create on first real
  pipeline run.

### `community.wave.seqera.io` (Seqera Wave multi-tool containers) — static mirror
Pull-through cache doesn't support this registry at all (not in AWS's supported
upstream list: `ecr`, `ecr-public`, `quay`, `k8s`, `docker-hub`,
`github-container-registry`, `azure-container-registry`, `gitlab-container-registry`,
`chainguard` — confirmed by testing, API rejects it outright). Instead, 22 unique
images were `docker pull`/`tag`/`push`ed one-time into static ECR repos under
`community/library/*` in `677276073900` (repos created + repository policy
applied manually per-repo first, since the repo-creation-template mechanism only
applies to PULL_THROUGH_CACHE/REPLICATION-created repos, not manually created
ones). All 22/22 verified present with correct tags. These don't auto-update if
a module's container tag changes upstream — re-run the mirror step manually if
that happens.

### Repository policy, applied to all 37 repos (22 mirrors + 15 pre-existing
`quay/*` in the original `535002863113` setup)
Same two-statement grant as above (`omics.amazonaws.com` + cross-account pod
role) — every repo had **no policy at all** before this, so nothing could pull
from any of them regardless of IAM role permissions or whether the image was
mirrored.

### Workbench workspace registration
Registered as an external ECR resource in `bugbash-nextflowonaws`
(`wb resource create ecr-external-repository --account=677276073900
--region=us-east-1`, no `--repository-name`) so it wildcards every repo in that
account/region — covers both `quay/*` (lazily created) and `community/library/*`
(already all present) under one resource. Per the same Workbench doc, adding a
*second* external-ECR resource scoped to the same account/region "may not allow
access" — so `community/library/*` deliberately does **not** get its own
resource entry; it rides on the existing wildcard.

## 3. HealthOmics-specific Nextflow engine quirks discovered

- **`Files.isDirectory()` / `NoSuchKeyException` on new S3 prefixes.** HealthOmics's
  custom Nextflow S3 filesystem provider
  (`com.amazon.omics.engine.nextflow.common.file.s3.v2.S3FileSystemProviderV2Base`)
  doesn't catch `NoSuchKeyException` and translate it to "doesn't exist, return
  false" the way every other S3 NIO provider does — it lets the raw exception
  crash the run. Hit this at pipeline startup (`dumpParametersToJSON` writing
  `pipeline_info/params_*.json`) and again later (`collectFile` writing
  `pipeline_info/software_versions.yml`). Root cause was `outdir` being a raw S3
  URI; fixed by using `/mnt/workflow/pubdir` instead (§1). A local subworkflow
  patch (`utils_nfcore_rnaseq_pipeline/main.nf`, disabling `dump_parameters`) was
  tried as a workaround first and then reverted once the real fix was found — no
  net change to that file.
- **Definition zip size limit (100 MiB).** HealthOmics private workflow
  definitions can't exceed `104857600` bytes. Not actually caused by the
  S3-hosted workflow definition (only 18.5 MB, confirmed no `.git/` in it) — most
  likely from a local zip step operating on the full git checkout (103 MB, 83 MB
  of which is `.git/`). Any local zip/upload step for the workflow definition
  should exclude `.git/`, `docs/`, `tests/`, `.nextflow/`, and other non-runtime
  directories.

## Not yet resolved / worth tracking

- Whether HealthOmics's "container registry maps" feature
  (`registryMappings`/`imageMappings`, set at `create-workflow` time) is needed on
  top of the registry/repository policies above. Current theory: not needed,
  since our container references are already literal ECR URIs rather than
  `quay.io`/`community.wave.seqera.io` URIs — that feature exists to translate the
  latter automatically. Unconfirmed by a clean run yet.