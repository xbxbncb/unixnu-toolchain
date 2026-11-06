# unixnu-toolchain

Prebuilt Linux cross toolchain for [UniXNU](https://github.com/UniXNU/UniXNU):
LLVM built from source with the XNU builtin patch, plus cctools-port, `mig`,
`iig`, `lzfse` and the runner shims.

Building it takes 1-3 hours on a 4-vCPU runner, so it is built once here and
published as a release asset. Device repositories (UniXNU-mondrian and future
ports) download the asset instead of rebuilding it in every kernel run.

## Asset naming

```
unixnu-toolchain-ubuntu-24.04-<first 16 hex digits of sha256 over scripts/linux>.tar.zst
```

The hash covers every file under UniXNU core's `scripts/linux/`, so a toolchain
change (setup script, patches, pinned LLVM) always produces a new asset and the
device workflows never restore a stale binary. The pinned core commit is
`UNIXNU_REF` in `.github/workflows/toolchain.yml` and must match the device
workflow's pin for the names to line up.

## Building

Run the **Build cross toolchain** workflow. If the asset for the current
`scripts/linux` hash already exists the run stops after the lookup; pass
`force: true` to rebuild and clobber it.

The archive is `tar.zst` of `/opt/unixnu-linux` (numeric owners). It is about
0.5 GB, under GitHub's 2 GiB per-asset limit.

## Restoring in a device workflow

```yaml
- name: Restore published toolchain
  run: |
    asset=unixnu-toolchain-ubuntu-24.04-$(find UniXNU/scripts/linux -type f -print0 |
      sort -z | xargs -0 sha256sum | sha256sum | cut -c1-16).tar.zst
    gh release download unixnu-toolchain --repo xbxbncb/unixnu-toolchain \
      --pattern "$asset" --dir "$RUNNER_TEMP"
    zstd -dc "$RUNNER_TEMP/$asset" | sudo tar -C /opt -xf -
```

Then run `setup-toolchain.py` as usual: it is idempotent and only refreshes the
shims under `/usr/bin` and `CommandLineTools`, which live outside the prefix.

The repository is public so device workflows can fetch the asset with their own
`GITHUB_TOKEN`; nothing here is secret.
