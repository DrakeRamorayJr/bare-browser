# The patch series

The series in [`patches/`](../patches) is 406 files, `0000001.patch` to `0000460.patch`, numbered
in the order they apply. The files up to `0000428.patch` carry no descriptions. From `0000429.patch` on, each file
has a mail header with a subject and a description.

`0000210.patch` is not in the series. That number belongs to the Google Play build, which compiles
the update check out. The GitHub build keeps the check. `0000114.patch` and `0000117.patch` are also missing: Chromium 157
removed the code 0000117 changed, and Chromium 158 removed the old passwords row that 0000114 hid. `0000298.patch` to `0000323.patch` are not in this series either; those numbers belong to the 2.1 line.

From a clean Chromium checkout at the pinned revision, apply all of them in one command:

```bash
git apply --index patches/*.patch
```

The shell expands the glob in filename order, which is the order that matters. The patches are
staged and not committed. Use `git apply`, not `git am`, for the whole series: most files before
`0000429.patch` are plain diffs without a mail header, which `git am` rejects. The files from
`0000429.patch` on do have the header, so one of them can also be committed with `git am` on top of the
ones before it.
