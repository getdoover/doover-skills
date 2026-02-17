# Plan: Private App Support in Appgen

Status: **Draft — all open questions resolved, ready for implementation when needed**

## Overview

Currently appgen only creates public apps (visibility: PUB, ghcr.io registry, public GitHub repo). This plan covers what's needed to support private apps (visibility: PRI, DockerHub registry, private GitHub repo).

## Open Questions

- [x] **`container_registry_profile_key`**: **Resolved.** This is the Snowflake ID (big integer) of a `ContainerRegistryProfile` on the Doover platform. The profile must already exist before the app is published. The `ContainerRegistryProfile` model has no `key` slug field — only the `id`, so this integer ID is what goes in `doover_config.json`.
- [x] **Does the user know their profile ID?** They'll need to look it up in the Doover admin UI (Container Registries page). It's a one-time lookup per org since all private apps share the same DockerHub profile.
- [x] **DockerHub workflow template**: **Resolved.** Confirmed from `~/sia-injection-controller/.github/workflows/build-image.yml`. See "Confirmed DockerHub Workflow" section below. Note: the sia-injection-controller is a Doover 1.0-era app — some of its `doover_config.json` fields use UUIDs instead of Doover 2.0 Snowflake IDs. The workflow itself is fine to use as a template.

## Known Values

| Setting | Public Apps | Private Apps |
|---------|------------|--------------|
| `visibility` | `"PUB"` | `"PRI"` |
| Container registry | `ghcr.io/getdoover` | DockerHub (`spaneng`) |
| `image_name` | `ghcr.io/getdoover/{app}:main` | `spaneng/{app}:latest` |
| Image tag convention | `:main` (branch-based) | `:latest` (but see note — metadata-action auto-generates tags) |
| GitHub repo visibility | public | private |
| `owner_org_key` | optional (can be "FIX-ME") | **required** (user must provide) |
| `organisation_id` | optional | **required** (same as owner_org_key) |
| `container_registry_profile_key` | optional (null/empty) | **required** (Snowflake ID from Doover admin) |
| GitHub Actions auth | Built-in `GITHUB_TOKEN` | `DOCKERHUB_WRITE_USERNAME` (var) + `DOCKERHUB_WRITE_TOKEN` (secret) |
| DockerHub org | N/A | `spaneng` (always) |

## Changes Required

### 1. Phase 1 (phase-1-create.md)

**New question after app type (Question 4):**

Add a visibility question:
- "Should this app be public or private?"
- Options: "Public" / "Private"
- Store as `visibility: public` or `visibility: private`

**Conditional follow-up for private apps:**

If private, ask:
- "What is your Doover organisation key?" (required, no default)
- Store as `owner_org_key`

**Update GitHub repo creation (Step 4):**

```bash
# Currently:
gh repo create {org/repo-name} --source . --push --public

# For private apps:
gh repo create {org/repo-name} --source . --push --private
```

**Update PHASE.md state template:**

Add fields:
- `visibility: public|private`
- `owner_org_key: {key}` (only for private)
- `container_registry: ghcr.io/getdoover|spaneng`
- `container_registry_profile_key: {key}` (only for private, value TBD)

### 2. Phase 2 — All Variants (2d, 2p, 2w, 2i)

**Update doover_config.json generation:**

For private apps, set:
```json
{
    "visibility": "PRI",
    "image_name": "spaneng/{app-name}:latest",
    "owner_org_key": "{user-provided-key}",
    "organisation_id": "{user-provided-key}",
    "container_registry_profile_key": "{TBD}"
}
```

For public apps (unchanged):
```json
{
    "visibility": "PUB",
    "image_name": "ghcr.io/getdoover/{app-name}:main",
    "owner_org_key": "",
    "organisation_id": "",
    "container_registry_profile_key": ""
}
```

**Update `.github/workflows/build-image.yml` for private/DockerHub apps:**

The template workflow is GHCR-only. For DockerHub, it needs:
- Different auth (DockerHub secrets instead of GITHUB_TOKEN)
- Different image naming
- User needs to add `DOCKERHUB_WRITE_USERNAME` as a repo variable and `DOCKERHUB_WRITE_TOKEN` as a repo secret

**Confirmed DockerHub workflow** (from `~/sia-injection-controller`):
```yaml
name: Test, Lint & Deploy Docker Image

on:
  workflow_dispatch:
    inputs:
      confirmation:
        description: 'Push image on failed test or linting (not recommended!)?'
        required: true
        type: boolean

  push:
    branches:
      - main

env:
  REGISTRY: spaneng
  IMAGE_NAME: {app-name}

jobs:
  push_to_registries:
    name: Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@f4ef78c080cd8ba55a85445d5b36e214a81df20a
        with:
          username: ${{ vars.DOCKERHUB_WRITE_USERNAME }}
          password: ${{ secrets.DOCKERHUB_WRITE_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Push Images
        uses: docker/build-push-action@3b5e8027fcad23fda98b2e3ac259d8d67585f671
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Notes on this workflow:**
- `docker/metadata-action` auto-generates tags from branch name (so pushing to `main` produces `:main` tag)
- The `doover_config.json` `image_name` should use `:latest` to match the DockerHub convention, but the actual CI tag comes from metadata-action
- The sia-injection-controller reference also has lint + test jobs as prerequisites (reusable workflows), which appgen's existing GHCR template already follows — replicate the same pattern
- Removed `packages: write` permission (that's GHCR-specific, not needed for DockerHub)
- Action versions are pinned by commit hash (good practice, keep this)

### 3. Phase 5/Check — All Variants

Add a check that verifies:
- `visibility` matches expected value
- `image_name` uses correct registry prefix
- `owner_org_key` is not "FIX-ME" for private apps
- `.github/workflows/build-image.yml` matches the registry type

### 4. Summary/Documentation

After Phase 6 (Document), remind the user:
- For private apps: "You need to add `DOCKERHUB_WRITE_USERNAME` as a **repository variable** and `DOCKERHUB_WRITE_TOKEN` as a **repository secret** in your GitHub repo settings for CI/CD to work."

## How `doover app create` Works (Reference)

The CLI command (`~/doover-cli/src/doover_cli/apps.py`) is purely local:
1. Downloads `getdoover/app-template` tarball from GitHub
2. Extracts, renames files (PascalCase/snake_case/kebab-case)
3. Runs `uv run app_config.py` to export config schema
4. Updates doover_config.json with provided values
5. Git init + initial commit

**No API calls during creation.** The visibility, registry, and org values only matter at `doover app publish` time.

The CLI has a `ContainerRegistry` enum:
```python
class ContainerRegistry(Enum):
    GITHUB_INT = "ghcr.io/getdoover"
    GITHUB_OTHER = "ghcr.io/other"
    DOCKERHUB_INT = "DockerHub (spaneng)"
    DOCKERHUB_OTHER = "DockerHub (other)"
```

Image name is constructed as:
```python
f"{registry}/{name}:{'main' if registry.startswith('ghcr') else 'latest'}"
```

## pydoover Visibility Constants

```python
class Visibility:
    core = "COR"      # Internal Doover core apps
    public = "PUB"    # Available to all users
    private = "PRI"   # Only visible to owner/org
    internal = "INT"  # Internal to organization
```

## Implementation Order (When Ready)

1. Update Phase 1 — add visibility question + conditional org key question + private repo flag
2. Update all Phase 2 variants — conditional doover_config.json values + DockerHub workflow
3. Update all Phase 5/Check variants — validate visibility/registry consistency
4. Update Phase 6 — DockerHub secrets reminder in README
5. Test with a real private app creation

## Key Files

| File | Change Needed |
|------|---------------|
| `references/phase-1-create.md` | Add visibility question, org key question, private repo flag |
| `references/phase-2d-config.md` | Conditional doover_config.json + workflow |
| `references/phase-2p-config.md` | Conditional doover_config.json + workflow |
| `references/phase-2w-config.md` | Conditional doover_config.json (no workflow — widgets are zip, not Docker) |
| `references/phase-2i-config.md` | Conditional doover_config.json + workflow |
| `references/phase-5d-check.md` | Validate visibility/registry match |
| `references/phase-5p-check.md` | Validate visibility/registry match |
| `references/phase-4w-check.md` | Validate visibility (no Docker for widgets) |
| `references/phase-5i-check.md` | Validate visibility/registry match |
| `references/phase-6-document.md` | DockerHub secrets reminder for private apps |
| `~/doover-cli/src/doover_cli/apps.py` | No changes needed (appgen handles it post-creation) |

## Note on Widget Apps

Widget apps (and processor apps) deploy as **zip packages via Lambda**, not Docker containers. They don't have a `build-image.yml` workflow or `image_name`. For widgets/processors, private app support only requires:
- `visibility: "PRI"` in doover_config.json
- `owner_org_key` set to real value
- Private GitHub repo
- No Docker/registry changes needed

## Container Registry Profile — What It Is

**Source:** `Doover2.0_base/doover-control/doover_control/applications/models.py`

The `ContainerRegistryProfile` is a **Doover platform database model** that stores Docker registry credentials. It's how the platform authenticates when pulling container images during device deployments.

### Model Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SnowflakeID | Primary key / identifier |
| `organisation` | FK → Organisation | Scoped to an org (nullable for superuser) |
| `name` | CharField(255) | Display name, e.g. "DockerHub Prod" |
| `description` | TextField | Notes about scopes, usage |
| `url` | CharField(200) | Registry URL (DockerHub: `https://index.docker.io/v1/`, GitHub: `ghcr.io`) |
| `username` | CharField(100) | Registry username or org name |
| `password` | CharField(100) | Password or PAT |

### How It's Used in Deployment

When deploying a Docker app to a device:
1. Platform looks up `Application.container_registry_profile` (optional FK)
2. Extracts `url`, `username`, `password`
3. Sends credentials in the `docker_deployment` channel message to the device agent
4. Device agent uses credentials to `docker pull` the image

If the profile is `null`, the device pulls without auth (works for public images).

### API & Management

- **Endpoint:** `/container/registry/` (CRUD minus delete)
- **Admin UI:** `doover-admin/src/resources/conatiner_registry.tsx` (note: codebase has typo "conatiner" throughout)
- **Permissions:** `conatiner_registry_profile_view` and `conatiner_registry_profile_manage` on org roles

### Relationship to doover_config.json

The `container_registry_profile_key` field in `doover_config.json` is the **ID of an existing ContainerRegistryProfile on the platform**. It is NOT something appgen creates — the org admin must have already set up the profile via the Doover admin UI. When `doover app publish` runs, it links the app to that profile.

For **public apps** (ghcr.io/getdoover), this field can be null/empty — public images don't need auth.
For **private apps** (DockerHub/spaneng), this field must reference a valid profile with DockerHub credentials.

## Doover 1.0 vs 2.0 — What NOT to Copy

**Source:** `~/sia-injection-controller/doover_config.json` (reference private/DockerHub app, still in Doover 1.0 format)

The sia-injection-controller is a working DockerHub app but its `doover_config.json` uses Doover 1.0 conventions. Appgen must produce **Doover 2.0 format only**.

| Field | sia-injection (1.0) | Doover 2.0 (what appgen should produce) |
|-------|---------------------|------------------------------------------|
| `key` | `"321535af-3b80-..."` (UUID) | **Omit entirely** — Doover 2.0 ignores this field |
| `owner_org` | `"48f2d2c4-..."` (UUID) | Use `owner_org_key` with org key string (not UUID) |
| `container_registry_profile_key` | `"ee4568d0-..."` (UUID) | Snowflake ID (big integer, e.g. `7191234567890123456`) |
| `organisation_id` | not present | **Add** — same value as `owner_org_key` |

**The workflow files (`build-image.yml`, `run-linting.yml`, `run-tests.yml`) are NOT version-specific** — they're pure GitHub Actions and are safe to use as templates regardless of Doover version.

### Other observations from sia-injection-controller

- It's marked `visibility: "PUB"` but uses DockerHub (`spaneng/`) — so the public=GHCR / private=DockerHub mapping is not absolute. Some public apps may also use DockerHub. However, for appgen's purposes, the simple mapping (public→GHCR, private→DockerHub) is the right default.
- It uses `:main` tag in `image_name` despite being on DockerHub — the `docker/metadata-action` generates tags from branch name automatically. The `image_name` in doover_config.json should still use `:latest` for private apps per the CLI convention (`f"{registry}/{name}:{'main' if registry.startswith('ghcr') else 'latest'}"`).
- It has `build_args: "--platform linux/amd64,linux/arm64"` for multi-platform builds. The workflow handles this via `platforms: linux/arm64,linux/amd64` in the build step. Appgen should include multi-platform support in the DockerHub workflow template.
