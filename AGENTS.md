# Repository guidance

Keep this file focused on repository-wide workflow and conventions. Document
individual features in their patches, tests, and relevant product documentation.

## Layout

- This repository packages Proxmox VE's QEMU as `pve-qemu-kvm`.
- `qemu/` is the pinned upstream submodule. Device models, block storage,
  migration, tests, and firmware live under its corresponding source directories.
- `debian/patches/` contains quilt patches; `series` defines their application
  order. Preserve the existing `extra/`, `bitmap-mirror/`, and `pve/` organization.
- Root `Makefile` prepares sources and builds packages; `debian/rules` configures
  QEMU, installs binaries, and checks expected CPU flags and models.
- `debian/control` defines dependencies; `debian/changelog` defines the package
  version. `debian/parse-*.pl` generates capability data. `backup.txt` and
  `vma_spec.txt` describe backup behavior and the VMA format.

## Editing

- Inspect `git status` and submodule status first. Preserve unrelated changes and
  untracked files, and keep work within the requested scope.
- Maintain QEMU changes as patches in `debian/patches/` and register them in
  `series`. Include regression tests in the patch set. Changes left only in a
  prepared build tree will be lost; changes left only in the submodule are not
  included in the packaging repository's commit.
- Number repository-specific patches from `9000`; preserve upstream numbering.
- Update pinned submodule revisions as part of intended upstream updates, not
  as a workaround for patch failures. Verify the complete patch sequence applies.
- Follow surrounding QEMU C and Perl conventions. Run QEMU's
  `scripts/checkpatch.pl` on changed patches and review its findings; use
  `perl -c` for changed packaging helpers with their dependencies installed.
- Preserve QAPI, machine-version, migration, saved-state, and disk-format
  compatibility. Cover both configured targets, x86_64 and aarch64, where affected.
- Keep build trees, packages, logs, temporary images, and IDE files out of
  commits. Preserve tracked fixtures and capability baselines.
- Prepare Linux/WSL source copies with LF line endings and Git executable modes.

## Build and validation

Keep the existing package version when building or rebuilding. Change version
numbers or add suffixes only when explicitly requested; a request for a new
`.deb` does not authorize a version change. Same-named packages may be replaced.

Run builds in Linux, WSL, or an isolated Debian/PVE container with dependencies
from `debian/control`. When using `makedie/proxmox_ve:<tag>`, select a compatible
tag and verify its dependencies. Override its systemd entrypoint with
`--entrypoint /bin/bash`; use `-lc` for scripted commands.

From the repository root:

```sh
make submodule  # Initialize missing recursive submodules and Meson subprojects.
make deb        # Build main/debug packages and run lintian; also the default make target.
```

- Ensure recursive submodules and Meson subprojects are available before
  configuring: `debian/rules` disables dependency downloads. `make submodule`
  skips initialization when `qemu/configure` already exists.
- Keep `qemu/` unconfigured. Root source preparation replaces `BUILDDIR`, so
  persist changes as patches before rebuilding. Root `make clean` removes build
  trees and package artifacts.
- There is no root `test` or `check` target. Run applicable QEMU `check-unit`,
  `check-qtest`, and block iotests from the configured build tree. Packaging does
  not run the full QEMU test suite; select regressions appropriate to the change.
- For standalone qtests, set `QTEST_QEMU_BINARY` to the intended built emulator.
  For firmware boot checks, select matching firmware explicitly, for example
  with QEMU's `-L` option. Verify firmware source changes reach the packaged blobs;
  editing `roms/` alone can leave prebuilt `pc-bios/` files unchanged.
- Review CPU-flag and CPU-model baseline differences during packaging. Update
  expected files only for intentional changes, not to hide environmental failures.
- Verify package metadata and affected installed files. Report which emulator,
  firmware, and architectures were actually exercised; package checks do not
  prove behavior on a live PVE host.
- Finish with `git diff --check`. Documentation-only changes need diff review,
  not a QEMU build or runtime tests.

## Commit messages

- Use a concise Chinese subject in `<type>: <中文改动摘要>` form, followed by a
  brief Chinese overview explaining the resulting behavior and reason.
- Describe every changed file, including patches, tests, and documentation:
  put its repository-relative path on its own line, followed by bullets in the
  form `- <改动主题> 新增/修改/修复/删除：<具体变化>`.
- Omit `Signed-off-by:` trailers; do not use `git commit -s` or `--signoff`.
- Keep validation results in the report or PR, without a separate validation
  section in the commit message.
