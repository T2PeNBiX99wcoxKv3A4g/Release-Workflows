# Release Workflows

A collection of reusable [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) that automate **versioning, building and releasing** projects across multiple languages.

The workflows are modular: the version bump and the GitHub Release creation are shared by every language, while each language only provides its own build recipe.

- JVM (Gradle / Maven)
- Python (uv + cx_Freeze)
- C# (.NET)

## Module map

| Workflow                      | Kind    | Purpose                                                              |
|-------------------------------|---------|----------------------------------------------------------------------|
| `version_change.yml`          | shared  | Bump the version in a file, generate `CHANGELOG.md`, commit and push |
| `create_release.yml`          | shared  | Download build artifacts and create a GitHub Release                 |
| `jvm_build.yml`               | build   | Build a Gradle project and optionally publish a Maven repository     |
| `jvm_publish_maven_pages.yml` | extra   | Publish the Maven repository and deploy GitHub Pages                 |
| `jvm_rebuild_maven_pages.yml` | extra   | Rebuild the Maven Pages site without a new release                   |
| `setup_maven_pages.yml`       | extra   | Initialise / refresh the `maven` branch used by the Pages site       |
| `python_uv_build_linux.yml`   | build   | Build a Linux executable with cx_Freeze and archive it               |
| `python_uv_build_windows.yml` | build   | Build a Windows executable with cx_Freeze and archive it             |
| `jvm_release.yml`             | release | Orchestrates a full JVM release                                      |
| `python_uv_release.yml`       | release | Orchestrates a full Python release                                   |
| `csharp_release.yml`          | release | Orchestrates a full C# release                                       |
| `csharp_build.yml`            | build   | Publish a .NET project per runtime and archive it                    |
| `csharp_library_release.yml`  | release | Orchestrates a .NET library release (NuGet + raw DLL)                |
| `csharp_library_build.yml`    | build   | Pack, publish and upload a .NET library                              |
| `npm_release.yml`             | release | Orchestrates an npm package release                                  |
| `npm_build.yml`               | build   | Pack, publish and upload an npm package                              |
| `csharp_npm_release.yml`      | release | Orchestrates a combined .NET library + npm release                   |

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

When `publish_maven` is `true` the Maven repository is published to the `maven` branch and deployed to GitHub Pages.

### Python (uv) — `python_uv_release.yml`

| Input               | Type    | Default            | Description                 |
|---------------------|---------|--------------------|-----------------------------|
| `filename`          | string  | *(required)*       | Output archive prefix       |
| `version_file_path` | string  | `./pyproject.toml` | File that holds the version |
| `extra_version_files` | string | `''`             | Newline-separated files kept in sync with `sync-version`; append `\|<key>` to limit to a key |
| `version_keyword`   | string  | `version`          | Version key                 |
| `set_version_max_matches` | number | `1` | How many matching lines set-version may rewrite (1 = only the authoritative version; -1 = every match) |
| `prerelease`        | boolean | `false`            | Produce a pre-release       |

Builds on both `ubuntu-latest` and `windows-latest` and releases per-OS `<filename>-Linux.zip` / `<filename>-Windows.zip` archives.

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
    uses: T2PeNBiX99wcoxKv3A4g/Release-Workflows/.github/workflows/jvm_rebuild_maven_pages.yml@main
    secrets: inherit
```

## Notes

- **Secrets** — always add `secrets: inherit` when calling these workflows. `create_release` and `version_change` rely on `GITHUB_TOKEN`.
- **Permissions** — calling jobs that push commits, tags or Pages need `contents: write` (and `pages: write` / `id-token: write` for the Maven Pages workflows). Grant them in the caller when the default token is restricted.
- **Artifacts** — the release step downloads every artifact of the run and merges them into `release/`; each language names its artifacts so files never clash (JVM ships `*.jar`, Python and C# ship per-platform `*.zip`).
