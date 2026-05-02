# GitHub action to run Dagger

## Usage Examples

### `dagger check`

```yaml
- name: Hello
  uses: dagger/dagger-for-github@v8.4.0
  with:
    check: "**"
    version: "latest"  # semver vX.Y.Z
```

Note: As a convenience for `dagger check` you can use the [dagger/checks](https://github.com/dagger/checks) action instead.

### `dagger call`

```yaml
- name: Hello
  uses: dagger/dagger-for-github@v8.4.0
  with:
    module: github.com/shykes/daggerverse/hello
    call: hello --greeting Hola --name Jeremy
    cloud-token: ${{ secrets.DAGGER_CLOUD_TOKEN }}
    version: "latest"  # semver vX.Y.Z
```

### `dagger shell`

```yaml
- name: Hello
  uses: dagger/dagger-for-github@v8.4.0
  with:
    shell: container | from alpine | with-exec echo,"hello, world!" | stdout
    cloud-token: ${{ secrets.DAGGER_CLOUD_TOKEN }}
```

### `dagger run`

```yaml
- name: Integration Test
  uses: dagger/dagger-for-github@v8.4.0
  with:
    workdir: db-service
    verb: run
    args: node build.js
    cloud-token: ${{ secrets.DAGGER_CLOUD_TOKEN }}
    version: "latest"  # semver vX.Y.Z
```

### Staying in sync with the `latest` version

By setting the version to `latest`, this action will install the latest version of Dagger.

### Persisting the Dagger engine cache across runs

Set `cache-engine: true` to persist the engine's `/var/lib/dagger` state (BuildKit layers, module cache) between workflow runs via [`actions/cache`](https://github.com/actions/cache). The action also caches the engine container image itself, so warm runs skip the `docker pull` from `registry.dagger.io`.

```yaml
- name: Hello
  uses: dagger/dagger-for-github@v8.4.0
  with:
    call: hello --greeting Hola --name Jeremy
    cache-engine: true
    # Optional: tie cache invalidation to your source files
    cache-engine-key: dagger-engine-${{ hashFiles('**/go.sum') }}
```

How it works:

- Two cache layers are managed: the engine **image** (keyed on dagger version, shared across all jobs) and the engine **state** (keyed per-job, per-workflow, per-ref, per-sha with progressive fallbacks).
- The action pre-starts the engine container with the state directory bind-mounted, points the CLI at it via `_EXPERIMENTAL_DAGGER_RUNNER_HOST`, and gracefully stops it (60s timeout) so BuildKit flushes state to disk.
- State is packed into a single zstd tarball before saving — BuildKit's overlay snapshot work directories use modes the unprivileged runner user cannot read directly, so `sudo tar` bridges the directory and a runner-readable archive.
- Saves are skipped when the restore was an exact primary-key hit, to avoid the misleading "another job may be creating this cache" warning.

### All `with:` input parameter options

| Key                             | Description                                                       | Required | Default            |
| ------------------------------- | ----------------------------------------------------------------- | -------- | ------------------ |
| `version`                       | Dagger Version. Use semver vX.Y.Z or 'latest'                     | true     | 'latest'           |
| `commit`                        | Dagger Dev Commit (overrides `version`)                           | false    | ''                 |
| `dagger-flags`                  | Dagger CLI Flags                                                  | false    | '--progress plain' |
| `verb`                          | CLI verb (call, run, download, up, functions, shell, query)       | false    | 'call'             |
| `workdir`                       | The working directory in which to run the Dagger CLI              | false    | '.'                |
| `cloud-token`                   | Dagger Cloud Token                                                | false    | ''                 |
| `module`                        | Dagger module to call. Local or Git                               | false    | ''                 |
| `args`                          | Arguments to pass to CLI                                          | false    | ''                 |
| `call`                          | Arguments to pass to CLI (Alias for args with verb:call)          | false    | ''                 |
| `shell`                         | Arguments to pass to CLI (Alias for args with verb:shell)         | false    | ''                 |
| `check`                         | Arguments to pass to CLI (Alias for args with verb:check)         | false    | ''                 |
| `summary-path`                  | File path to write the job summary to                             | false    | ''                 |
| `enable-github-summary`         | Whether to automatically write a GitHub Actions job summary       | false    | 'false'            |
| `cache-engine`                  | Persist the engine's `/var/lib/dagger` state across runs          | false    | 'false'            |
| `cache-engine-key`              | Prefix for the actions/cache key used by `cache-engine`           | false    | 'dagger-engine'    |
| `cache-engine-archive`          | Host path for the engine state archive (defaults to a tarball under `RUNNER_TEMP`) | false    | ''                 |

### All output variables

| Key        | Description                                                 |
| ---------- | ----------------------------------------------------------- |
| `stdout`   | The standard output of the Dagger command                   |
| `traceURL` | Dagger Cloud trace URL                                      |
