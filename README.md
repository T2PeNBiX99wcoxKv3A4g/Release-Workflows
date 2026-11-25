# Release Workflows

A collection of reusable [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) that automate **versioning, building and releasing** projects across multiple languages.

The workflows are modular: the version bump and the GitHub Release creation are shared by every language, while each language only provides its own build recipe.

- JVM (Gradle / Maven)
- Python (uv + cx_Freeze)
- C# (.NET)
- Blender extensions (built with the Blender CLI)

## Module map

| Workflow                      | Kind    | Purpose                                                              |
|-------------------------------|---------|----------------------------------------------------------------------|
| `version_change.yml`          | shared  | Bump the version in a file, generate `CHANGELOG.md`, commit and push |
| `changelog.yml`               | shared  | Regenerate `CHANGELOG.md` for an already tagged version, commit and push |
| `create_release.yml`          | shared  | Download build artifacts and create a GitHub Release                 |
| `jvm_build.yml`               | build   | Build a Gradle project and optionally publish a Maven repository     |
| `jvm_maven_pages.yml`         | extra   | Publish the Maven repository and deploy the Pages site               |
| `python_uv_build.yml`         | build   | Build the Linux and Windows executables with cx_Freeze and archive them |
| `jvm_release.yml`             | release | Orchestrates a full JVM release                                      |
| `python_uv_release.yml`       | release | Orchestrates a full Python release                                   |
| `csharp_release.yml`          | release | Orchestrates a full C# release                                       |
| `csharp_build.yml`            | build   | Publish a .NET project per runtime and archive it                    |
| `csharp_library_release.yml`  | release | Orchestrates a .NET library release (NuGet + raw DLL)                |
| `csharp_library_build.yml`    | build   | Pack, publish and upload a .NET library                              |
| `npm_release.yml`             | release | Orchestrates an npm package release                                  |
| `npm_build.yml`               | build   | Pack, publish and upload an npm package                              |
| `csharp_npm_release.yml`      | release | Orchestrates a combined .NET library + npm release                   |
| `blender_extension_build.yml` | build   | Build a Blender extension ZIP with the Blender CLI                   |
| `blender_extension_release.yml` | release | Orchestrates a full Blender extension release                      |
| `vpm_build.yml`               | build   | Build a VPM package `.zip` + `.unitypackage`                         |
| `vpm_release.yml`             | release | Orchestrates a full VPM package release                              |
| `vpm_listing.yml`             | extra   | Build the listing frontend and the VPM package listing, deploy to Pages |
| `web_pages.yml`               | extra   | Build a Node site and deploy it to GitHub Pages                      |

## How it works

Every `*_release.yml` orchestrator follows the same shape:

```
                    +-------------------+
                    |  version_change   |  bump version, changelog, tag
                    +---------+---------+
                              | new_tag / changelog
              +---------------+---------------+
              v               v               v
       +-------------+ +-------------+ +-------------+
       |  jvm_build  | | python_uv   | | csharp_     |  build + upload artifacts
       |             | | _build_*    | | build       |
       +------+------+ +------+------+ +------+------+
              +---------------+---------------+
                              v
                    +-------------------+
                    |  create_release   |  attach artifacts to the release
                    +-------------------+
```

- `version_change` runs first and outputs `new_tag`, `new_version`, `changelog`, `previous_tag` and `release_type`.
- The build job starts only after `version_change` (`needs: [ version-change ]`) so it builds the already-bumped source.
- `create_release` runs last and consumes the tag, the changelog and the uploaded artifacts.

> Reusable workflows are called with `uses:` and **must** forward secrets with `secrets: inherit`, otherwise `GITHUB_TOKEN` (and any custom secret) is not available inside them.

## Shared workflows

### `version_change.yml`

Bumps the version using [`mathieudutour/github-tag-action`](https://github.com/mathieudutour/github-tag-action), updates the version file with [`Version-Change-Workflow-Scripts/change_version.py`](https://github.com/T2PeNBiX99wcoxKv3A4g/Version-Change-Workflow-Scripts), regenerates the changelog with [git-cliff](https://github.com/orhun/git-cliff-action) and commits the result.

The changelog half (git-cliff, commit, push, release body) is the
[`changelog` composite action](.github/actions/changelog/action.yml), so this workflow
and `changelog.yml` share one implementation instead of two copies. It is referenced by
repository — `T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/actions/changelog@main` — not as
`./.github/actions/changelog`: a called workflow resolves a local action against the **caller's**
checkout, so the relative form works only while these workflows run inside this repository.

If the tag action produces no new tag — for example when a workflow is retried and there are no new commits since the last tag — the latest existing tag is reused, so the version bump and release can still run.

| Input                  | Type    | Default                                   | Description                                                                                           |
|------------------------|---------|-------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `version_file_path`    | string  | *(required)*                              | File that holds the version (e.g. `gradle.properties`, `pyproject.toml`, `Directory.Build.props`)     |
| `version_keyword`      | string  | `version`                                 | Exact key name to match (case-sensitive)                                                              |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match)                                   |
| `extra_version_files` | string | `''`                                   | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key (e.g. `package.json\|version`) |
| `update_readme`        | boolean | `false`                                   | Also sync the version inside the README                                                               |
| `readme_file_path`     | string  | `./README.md`                             | README file to update                                                                                 |
| `require_keywords`     | string  | `''`                                      | Comma separated keywords the target line must **also** contain (`set-version --require-keywords`)  |
| `replace_version_key`  | string  | `''`                                      | Limit `sync-version` to lines whose key matches; empty replaces every occurrence (`--version-key`) |
| `debug`                | boolean | `false`                                   | Enable `icecream` debug output of the scripts                                                         |
| `prerelease`           | boolean | `false`                                   | Produce a pre-release tag                                                                             |
| `commit_message`       | string  | `action: Update version and CHANGELOG.md` | Commit message                                                                                        |
| `git_cliff_config_url` | string  | git-cliff template URL                    | Changelog configuration                                                                               |
| `python_scripts_repo`  | string  | `Version-Change-Workflow-Scripts` URL     | Repository cloned for the Python helper scripts                                                       |

| Output         | Description                                           |
|----------------|-------------------------------------------------------|
| `new_tag`      | The tag that was created                              |
| `new_version`  | The new version (normalised, without the leading `v`) |
| `changelog`    | The generated release body (latest section)           |
| `previous_tag` | The tag before the bump                               |
| `release_type` | Detected bump type (major / minor / patch)            |

### `changelog.yml`

Keeps `CHANGELOG.md` in sync for projects that version themselves in their own
manifest instead of letting `version_change` bump it. It regenerates the changelog
from the tags already in the repository, commits the result, pushes it and returns
the release body for the latest tag.

| Input                  | Type    | Default                    | Description                                                     |
|------------------------|---------|----------------------------|-----------------------------------------------------------------|
| `git_cliff_config_url` | string  | git-cliff template URL     | Changelog configuration                                         |
| `changelog_file`       | string  | `CHANGELOG.md`             | File that is regenerated and committed                          |
| `commit_message`       | string  | `action: Update CHANGELOG.md` | Commit message                                               |
| `ref`                  | string  | `''`                       | Branch to commit to (default: the ref of the caller)            |

| Output      | Description                                           |
|-------------|-------------------------------------------------------|
| `changelog` | Release body for the latest tag (header stripped)     |

### `create_release.yml`

| Input                | Type    | Default      | Description                           |
|----------------------|---------|--------------|---------------------------------------|
| `tag`                | string  | *(required)* | Tag for the release                   |
| `changelog`          | string  | `''`         | Release body                          |
| `artifacts`          | string  | `release/*`  | Artifact glob attached to the release |
| `prerelease`         | boolean | `false`      | Mark the release as a pre-release     |
| `release_name`       | string  | `''`         | Custom release name                   |
| `download_artifacts` | boolean | `true`       | Download build artifacts first        |
| `artifacts_path`     | string  | `release`    | Where artifacts are downloaded        |

## Language recipes

### JVM — `jvm_release.yml`

| Input                | Type    | Default               |
|----------------------|---------|-----------------------|
| `java_version`       | string  | `25`                  |
| `java_distribution`  | string  | `microsoft`           |
| `version_file_path`  | string  | `./gradle.properties` |
| `extra_version_files` | string | `''`                  | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key |
| `version_keyword`    | string  | `version`             |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match) |
| `update_readme`      | boolean | `false`               |
| `prerelease`         | boolean | `false`               |
| `publish_maven`      | boolean | `false`               |
| `maven_branch`       | string  | `maven`               |
| `maven_template_url` | string  | Maven template URL    |
| `artifacts_path`     | string  | `build/libs/`         |
| `maven_repo_path`    | string  | `build/repo/`         |

When `publish_maven` is `true` the Maven repository is tracked on its own branch
(`maven` by default), refreshed from `maven_template_url` when the branch is missing,
and the generated site is deployed to GitHub Pages. Called without a `tag` the same
workflow only rebuilds the Pages site.

### Python (uv) — `python_uv_release.yml`

| Input               | Type    | Default            | Description                 |
|---------------------|---------|--------------------|-----------------------------|
| `filename`          | string  | *(required)*       | Output archive prefix       |
| `version_file_path` | string  | `./pyproject.toml` | File that holds the version |
| `extra_version_files` | string | `''`             | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key |
| `version_keyword`   | string  | `version`          | Version key                 |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match) |
| `prerelease`        | boolean | `false`            | Produce a pre-release       |

Builds on both `ubuntu-latest` and `windows-latest` (one matrix in `python_uv_build.yml`) and releases per-OS `<filename>-Linux.zip` / `<filename>-Windows.zip` archives. The build workflow can also be called on its own:

```yaml
jobs:
  build:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/python_uv_build.yml@main
    with:
      filename: myapp
```

### Blender extension — `blender_extension_release.yml`

| Input               | Type    | Default                   | Description                                                                                       |
|---------------------|---------|---------------------------|---------------------------------------------------------------------------------------------------|
| `blender_version`   | string  | `5.3.0`                   | Blender release used to build (the builder version, not the supported range)                       |
| `source_dir`        | string  | `.`                       | Directory containing `blender_manifest.toml`                                                       |
| `version_file_path` | string  | `./blender_manifest.toml` | File that holds the extension version                                                              |
| `version_keyword`   | string  | `version`                 | Version key (exact, case-sensitive, token-bound)                                                   |
| `set_version_max_matches` | number | `1`                 | How many matching lines set-version may rewrite                                                    |
| `extra_version_files` | string | `''`                    | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key        |
| `prerelease`        | boolean | `false`                   | Produce a pre-release                                                                              |
| `tag_prefix`        | string  | `v`                       | Prefix expected on tags; the shared changelog template only recognises `v` tags                     |

The build downloads the official Blender Linux tarball outside the workspace, then runs
`blender --command extension build`. Everything inside the checkout would otherwise be
packed into the ZIP, so the Blender install is kept in `$RUNNER_TEMP`.

The manifest version is asserted against the tag from `version_change` before building. A
`workflow_call` build normally checks out the commit that triggered the run, which is the
pre-bump commit; without the assertion the ZIP would silently carry the previous version
number.

Version strings written into `blender_manifest.toml` must be semantic versions: Blender
accepts `1.2.3`, `1.2.3-beta.1` and `1.2.3+dev`, but rejects four-part versions such as
`1.2.3.4`.

### C# (.NET) — `csharp_release.yml`
| Input               | Type    | Default                    | Description                                                                                                             |
|---------------------|---------|----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| `dotnet_version`    | string  | `10.0.x`                   | SDK version                                                                                                             |
| `configuration`     | string  | `Release`                  | Build configuration                                                                                                     |
| `project_path`      | string  | `.`                        | Path to the `.csproj` (or a directory containing it). **Must not be a `.sln`**: `--output` is unsupported for solutions |
| `runtimes`          | string  | `["win-x64", "linux-x64"]` | JSON array of RIDs to publish for                                                                                       |
| `setup_aot_toolchain` | boolean | `false`                  | Install the Native AOT build dependencies of the target OS (only needed with `PublishAot=true`)                          |
| `filename`          | string  | `release`                  | Output archive prefix                                                                                                   |
| `version_file_path` | string  | `./Directory.Build.props`  | File that holds the version                                                                                             |
| `version_keyword`   | string  | `Version`                  | Version key (`<Version>` in MSBuild files)                                                                              |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match)                                   |
| `extra_version_files` | string | `''`                 | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key                        |
| `update_readme`     | boolean | `false`                    | Also sync the version inside the README                                                                                 |
| `readme_file_path`  | string  | `./README.md`              | README file to update                                                                                                   |
| `prerelease`        | boolean | `false`                    | Produce a pre-release                                                                                                   |
| `msi_script`        | string  | `''`                       | Script run on Windows after publishing to build an installer; it reads `VERSION`, `PUBLISH_DIR`, `OUTPUT_DIR` and `RUNTIME` from the environment. Empty builds no MSI |
| `rpm_script`        | string  | `''`                       | Script run on Linux inside a Fedora container after publishing to build an RPM; it reads the same environment. Empty builds no RPM |

Each runtime is published with `dotnet publish -r <rid> -c <configuration>` and packaged into `<filename>-<rid>.zip`, so the archives never collide when the release merges them.

Packaging is left to the project, because only it knows where its files belong. Point `msi_script` at a script that builds an installer into `OUTPUT_DIR`, or `rpm_script` at one that builds an RPM there; anything matching `*.msi` or `*.rpm` in that directory is attached to the release alongside the archive. The RPM script runs inside `fedora:latest` with `rpmbuild` preinstalled and the repository mounted at `/src`:

```yaml
      msi_script: 'installer/windows/build-msi.ps1'
      rpm_script: 'installer/linux/build-rpm.sh'
```

Both scripts receive these variables:

| Variable | Value |
|---|---|
| `VERSION` | `<Version>` read from the project file after the version bump |
| `PUBLISH_DIR` | Directory the runtime was published to, relative to the repository |
| `OUTPUT_DIR` | Where the package should be written; same as `PUBLISH_DIR` here |
| `RUNTIME` | The RID being built, e.g. `win-x64` |

Each runtime runs on the runner of its own OS (`win*` → `windows-latest`, `linux*` → `ubuntu-latest`, `osx*` → `macos-latest`), which is required for Native AOT since cross-OS compilation is not supported.

> `setup_aot_toolchain` is `false` by default because most projects are not natively compiled. Set it to `true` when the project uses `PublishAot=true`; the Linux job then installs `clang` and `zlib1g-dev` before publishing (Windows and macOS only need the shipped MSVC / clang toolchain).

> `project_path` accepts either the `.csproj` file itself or the directory that contains it. Pointing it at a directory that only holds a `.sln` fails with `NETSDK1194`, because a solution-level `--output` is not supported.

### C# library (.NET) - `csharp_library_release.yml`

For class libraries that ship a NuGet package. The build produces
`<PackageId>.<version>.nupkg` and the raw `<project>.dll`; both are attached to the GitHub Release.
npm is a separate workflow: use `npm_release.yml` on its own, or `csharp_npm_release.yml` when the
release carries both.

| Input | Type | Default | Description |
|---|---|---|---|
| `project_path` | string | *(required)* | `.csproj` or the directory containing it |
| `version_file_path` | string | *(required)* | File that holds the library version |
| `version_keyword` | string | `Version` | Version key in the primary file |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match) |
| `extra_version_files` | string | `''` | Newline-separated files synced with `sync-version`; append `\|<key>` |
| `publish_nuget` | boolean | `false` | Publish the `.nupkg` when `NUGET_API_KEY` is configured |
| `filename` | string | `library` | Build artifact name |
| `artifacts_path` | string | `release` | Directory where the artifacts are collected |
| `prerelease` | boolean | `false` | Create a pre-release |
| `dotnet_version` | string | `10.0.x` | .NET SDK version |
| `configuration` | string | `Release` | Build configuration |
| `update_readme` | boolean | `false` | Also sync the version inside the README |
| `readme_file_path` | string | `./README.md` | README to update |
| `commit_message` | string | `action: Update version and update CHANGELOG.md` | Version bump commit message |

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/csharp_library_release.yml@main
    with:
      project_path: ./src/MyLib/MyLib.csproj
      version_file_path: ./src/MyLib/MyLib.csproj
      publish_nuget: true
    secrets: inherit
```

### npm - `npm_release.yml`

Packs an npm package, optionally publishes it and attaches the tarball to the release.

| Input | Type | Default | Description |
|---|---|---|---|
| `npm_directory` | string | *(required)* | Directory of the npm package |
| `version_file_path` | string | *(required)* | File that holds the package version |
| `version_keyword` | string | `version` | Version key in the version file |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match) |
| `extra_version_files` | string | `''` | Newline-separated files synced with `sync-version`; append `\|<key>` |
| `npm_pack_command` | string | `npm run pack` | Command that builds and packs the package |
| `publish_npm` | boolean | `false` | Publish when `NPM_TOKEN` is configured |
| `node_version` | string | `22` | Node.js version |
| `filename` | string | `npm-package` | Build artifact name |
| `artifacts_path` | string | `release` | Directory where the tarball is collected |
| `prerelease` | boolean | `false` | Create a pre-release; npm uses the `next` tag |
| `commit_message` | string | `action: Update version and update CHANGELOG.md` | Version bump commit message |

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/npm_release.yml@main
    with:
      npm_directory: ./UserInterface
      version_file_path: ./UserInterface/package.json
      publish_npm: true
    secrets: inherit
```

### C# library + npm - `csharp_npm_release.yml`

One release for a library that ships both a NuGet package and an npm package. It combines the
inputs from the two sections above (`npm_directory` is required) and runs `csharp_library_build.yml`
and `npm_build.yml` side by side before creating the release.

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/csharp_npm_release.yml@main
    with:
      project_path: ./src/MyLib/MyLib.csproj
      version_file_path: ./src/MyLib/MyLib.csproj
      extra_version_files: |
        UserInterface/package.json|version
      npm_directory: ./UserInterface
      publish_nuget: true
      publish_npm: true
    secrets: inherit
```

## VPM packages

A VPM package repository keeps one Unity project with the package under
`Packages/<package_name>`, bumps `Packages/<package_name>/package.json` by hand and
releases it as a tag of the bare version (`1.2.3`, no `v` prefix — that is what the
VPM changelog template and the VCC resolver expect).

The repositories no longer carry a `Website/` directory: the listing frontend lives in
[`VPM-WebSite`](https://github.com/T2PeNBiX99wcoxKv3A4g/VPM-WebSite) and is built by
`vpm_listing.yml` into a throw-away `Website/` directory before publishing.

`vpm_release.yml` turns that bump commit into a release: it tags the version already
committed to the manifest, builds the package, updates `CHANGELOG.md` and attaches
everything to the GitHub Release.

```
+--------------+     +--------------+     +--------------+
|     tag      |---->|  vpm_build   |---->|              |
| package.json |     | zip + unityp |     | create_      |
+------+-------+     +--------------+     | release      |
       |                                   |              |
       +------------>+--------------+---->|              |
                     |  changelog   |     +--------------+
                     +--------------+
```

### `vpm_release.yml`

| Input                  | Type    | Default                   | Description                                                              |
|------------------------|---------|---------------------------|--------------------------------------------------------------------------|
| `package_name`         | string  | *(required)*              | VPM package id, which is also the folder name under `Packages/`           |
| `package_path`         | string  | `''`                      | Package directory; defaults to `Packages/<package_name>`                  |
| `tag_prefix`           | string  | `''`                      | Prefix of the tag. Empty tags the bare version                          |
| `prerelease`           | boolean | `false`                   | Create a pre-release                                                      |
| `changelog_file`       | string  | `CHANGELOG.md`            | Changelog file that is regenerated and committed                          |
| `commit_message`       | string  | ``Action: Update `CHANGELOG.md` `` | Commit message                                                    |
| `git_cliff_config_url` | string  | `cliff-vpm.toml` template | Changelog configuration                                                  |

| Output      | Description                                  |
|-------------|----------------------------------------------|
| `tag`       | Tag the release was created for              |
| `version`   | Version taken from `package.json`            |
| `changelog` | Generated release body                       |

The tag is created before the build, so a re-run of the same version rebuilds and
re-attaches the artifacts of that tag instead of inventing a new version.

### `vpm_build.yml`

Builds the two files a VPM package ships: `<package_name>-<version>.zip` (the package
folder contents) and `<package_name>-<version>.unitypackage` (built from the package
`.meta` files). `package.json` is collected next to them so the release carries the
manifest too. Call it directly when only the artifacts are needed.

| Input            | Type   | Default   | Description                                  |
|------------------|--------|-----------|----------------------------------------------|
| `package_name`   | string | *(required)* | VPM package id / folder name under `Packages/` |
| `package_path`   | string | `''`      | Package directory (default `Packages/<package_name>`) |
| `ref`            | string | `''`      | Ref to build (default: the ref of the caller) |
| `artifacts_path` | string | `release` | Where the artifacts are collected            |
| `artifact_name`  | string | `release` | Artifact name used for the upload            |

| Output    | Description                       |
|-----------|-----------------------------------|
| `version` | Version read from `package.json`  |

### `vpm_listing.yml`

Builds the VPM listing website with
[`vrchat-community/package-list-action`](https://github.com/vrchat-community/package-list-action)
and deploys it to GitHub Pages. Triggers stay in the caller, because a reusable
workflow cannot declare `workflow_run` / `release` events.

The frontend is not part of the package repositories: it is built from
[`VPM-WebSite`](https://github.com/T2PeNBiX99wcoxKv3A4g/VPM-WebSite) and only the package
data comes from the repository that runs the build.

```text
VPM-WebSite (npm ci && npm run build)
        |
        v
     dist/  ->  Website/  ->  package-list-action  ->  GitHub Pages
                                   ^
                                   |
             source.json / Packages/<name>/package.json
```

`dist/app.js` is a Scriban template, so the action can still inject the package list; this
is the only coupling left with `package-list-action`.

| Input                  | Type   | Default                              | Description                                                                     |
|------------------------|--------|--------------------------------------|---------------------------------------------------------------------------------|
| `publish_directory`    | string | `Website`                            | Directory the listing is generated into                                          |
| `site_repository`      | string | `T2PeNBiX99wcoxKv3A4g/VPM-WebSite`   | Repository that holds the listing frontend                                       |
| `site_ref`             | string | `main`                               | Ref of the listing frontend repository                                           |
| `site_directory`       | string | `.listing-site`                      | Directory the listing frontend is checked out into                               |
| `node_version`         | string | `22`                                 | Node.js version used to build the listing frontend                               |
| `ci_directory`         | string | `ci`                                 | Directory the automation repository is checked out into                          |
| `ci_repository`        | string | `vrchat-community/package-list-action` | Repository that provides the listing build                                     |
| `current_package_name` | string | `''`                                 | Package id this repository publishes; empty for a repository that only aggregates others |

The job needs `pages: write` and `id-token: write`, so the caller has to grant them.

## Usage

Call a workflow from any repository with `owner/repo/.github/workflows/<file>@<ref>`.
The easiest entry points are the `*_release.yml` orchestrators.

### JVM release

```yaml
name: Release

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/jvm_release.yml@main
    with:
      version_file_path: ./gradle.properties
      publish_maven: true
    secrets: inherit
```

### Python release

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/python_uv_release.yml@main
    with:
      filename: myapp
      version_file_path: ./pyproject.toml
    secrets: inherit
```

### C# release

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/csharp_release.yml@main
    with:
      project_path: ./src/MyApp/MyApp.csproj
      filename: MyApp
      runtimes: '["win-x64", "linux-x64"]'
    secrets: inherit
```

### Blender extension release

```yaml
jobs:
  release:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/blender_extension_release.yml@main
    with:
      blender_version: '5.3.0'
      version_file_path: ./blender_manifest.toml
      extra_version_files: |
        ./__init__.py|CATS_VERSION
    secrets: inherit
```

### VPM package release

The `on:` block stays in the package repository, because only it knows which manifest
triggers a release. The job itself is one `uses:` call:

```yaml
name: Build Release

on:
  workflow_dispatch:
  push:
    paths:
      - 'Packages/io.github.ykysnk.example/package.json'

jobs:
  release:
    permissions:
      contents: write
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/vpm_release.yml@main
    with:
      package_name: io.github.ykysnk.example
    secrets: inherit
```

### VPM listing

```yaml
name: Build Repo Listing

on:
  workflow_dispatch:
  workflow_run:
    workflows: [ Build Release ]
    types: [ completed ]
  release:
    types: [ published, created, edited, unpublished, deleted, released ]

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build-listing:
    permissions:
      contents: read
      pages: write
      id-token: write
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/vpm_listing.yml@main
    with:
      current_package_name: io.github.ykysnk.example
```

A repository that only aggregates other repositories (a `source.json` with a
`githubRepos` list) omits `current_package_name` and triggers on that file instead.

### Build only (no release)

Call a build workflow directly when you only need the artifacts:

```yaml
jobs:
  build:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/csharp_build.yml@main
    with:
      project_path: ./src/MyApp/MyApp.csproj
```

```yaml
jobs:
  build:
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/jvm_build.yml@main
    with:
      publish_maven: false
```

### Rebuild the Maven Pages site

```yaml
jobs:
  rebuild:
    permissions:
      contents: write
      pages: write
      id-token: write
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/jvm_maven_pages.yml@main
    secrets: inherit
```

## Notes

- **Secrets** — always add `secrets: inherit` when calling these workflows. `create_release` and `version_change` rely on `GITHUB_TOKEN`.
- **Permissions** — calling jobs that push commits, tags or Pages need `contents: write` (and `pages: write` / `id-token: write` for the Pages workflows). Grant them in the caller when the default token is restricted.
- **Artifacts** — the release step downloads every artifact of the run and merges them into `release/`; each language names its artifacts so files never clash (JVM ships `*.jar`, Python and C# ship per-platform `*.zip`).
