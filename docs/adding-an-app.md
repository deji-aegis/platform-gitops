# Adding a New Application

## Steps

1. Create the app directory under `applications/`:

   applications/<app-name>/
   ├── base/
   ├── dev/
   ├── staging/
   └── prod/

2. Copy the structure from `applications/wattzgoat/` as a template.

3. Update the app repo's CI to call `platform-ci/full-pipeline.yml`
   with the correct `image_name` and `gitops_repo`.

4. Open a PR against `platform-gitops`. CI will validate manifests.

5. Once merged, the ApplicationSet will auto-discover the new app
   within 3 minutes (default polling interval).
