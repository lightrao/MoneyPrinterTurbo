# Changelog

This changelog tracks the work on the
`feat/siliconflow-cloned-voice-v2` branch of `lightrao/MoneyPrinterTurbo`,
which is the fork deployed on HP450.

## Unreleased

- Switch the WebUI source dropdown to `private_catalog` on HP450 once the
  50-asset Catalog batch finishes acceptance.
- Promote the three new MPT integration tests to the regression matrix so
  any future `save_material` refactor keeps the read-key scoping guarantee.

## 2026-08-09 - Catalog staging snapshot (3-asset pilot accepted)

- `f23f3a1 docs(hp450): record three-asset catalog pilot`
- `c94e6e1 docs(hp450): record catalog release and blocked staging`
- `23b62de fix(material-catalog): scope catalog credentials to catalog downloads`
- `f9e8125 feat(material-catalog): add private catalog source with read-only shared volume`
- `2bc015b docs(hp450): record live operation baseline and WebUI limits`

Highlights:

- `private_catalog` provider wired into
  `app/services/material.py::save_material` with a `catalog_path` and a
  `/used` round-trip that records usage in the catalog so BM25 rotates.
- Read-only shared volume `${MPT_MATERIAL_VIDEO_DIR:-/data/projects/mpt-material-catalog/videos}:/MaterialCatalog/videos:ro`
  declared on `webui` and `api` in `deploy/hp450/compose.yml`.
- `external` network `mpt-catalog-net` declared on the MPT side so the
  Catalog project can be brought up independently.
- New `TestPrivateMaterialCatalog` cases: Pexels-shaped search response,
  Catalog read-key scoping (the regression that non-Catalog providers do
  **not** receive the Catalog `Authorization` header), read-only
  shared-volume path, catalog path traversal fallback, and `private_catalog`
  cache round-trip.
- HP450 staging: empty-catalog staging and a three-asset portrait pilot
  (12/13/13/12 plan reduced to 3 for the pilot) accepted on the live host;
  Pexels key cleared from the Catalog `.env` and the container
  environment. The full 50-asset batch and the MPT `private_catalog`
  cutover remain pending.
- Catalog repository `lightrao/MPTMaterialCatalog` is private; release
  `v0.1.2` manifest digest for `linux/amd64` is
  `sha256:33162f2e1d956c3745fb89d5b0db17c00ecb74ddb55b3d83d97dc6bd91e24ae7`.

## Earlier work in this fork

- `2bc015b docs(hp450): record live operation baseline and WebUI limits`
  added the live WebUI access section to `docs/deployment-hp450.md` and
  pinned the SSH-forward-only access pattern.
- `dc79217 docs: record HP450 image transfer fallback` documented the
  `crane pull --platform linux/amd64 --format tarball | ssh hp450 docker
  load` path used to bypass slow GHCR pulls.
- `a3bea03 ci: add reproducible HP450 container deployment` introduced
  the `Publish Docker image` workflow triggered on the
  `v1.3.3-hp450.*` tag pattern and the `sha-<commit>` tag convention.
- `1e6a32c test: cover SiliconFlow cloned voice flows` and earlier
  `3aaaa75 feat: add SiliconFlow cloned voice controls` plus
  `dcc28af feat: add SiliconFlow cloned voice dispatch` round out the
  SiliconFlow cloned voice feature work.