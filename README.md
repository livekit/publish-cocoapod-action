# publish-cocoapod-action

Reusable GitHub Action that publishes a podspec to a CocoaPods **specs repo** via PR.

LiveKit pods are not on the CocoaPods trunk — they live in [`livekit/podspecs`](https://github.com/livekit/podspecs),
which is a plain git repo, so a release is a **pull request**, not a `pod trunk push`.

The action clones the specs repo and hands it to `pod repo push --local-only`, so validation,
the `Specs/<Pod>/<version>/` layout and the `[Add]`/`[Update]` commit message are all CocoaPods'
own — the result is what a maintainer running `pod repo push` by hand would have produced.
It then pushes the branch and opens the PR.

## Usage

```yaml
jobs:
  publish:
    runs-on: macos-26
    steps:
      - uses: actions/checkout@v7
      - uses: maxim-lobanov/setup-cocoapods@v1
        with:
          version: latest
      - uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: latest-stable
      - uses: livekit/publish-cocoapod-action@<commit-sha> # main
        with:
          podspec: LiveKitClient.podspec
          token: ${{ secrets.PODSPECS_PAT }}
```

CocoaPods and Xcode are the caller's responsibility — validation builds the pod, so this needs a
macOS runner.

This action is not tagged — pin callers to a commit sha.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `podspec` | yes | — | Path to the `.podspec` file |
| `token` | yes | — | PAT with `contents: write` and `pull_requests: write` on the specs repo |
| `spec-repo` | no | `livekit/podspecs` | Target specs repo in `owner/name` format |
| `sources` | no | `https://github.com/livekit/podspecs.git/,https://cdn.cocoapods.org/` | Sources used to resolve dependencies while validating |
| `dry-run` | no | `false` | Validate and commit locally only — no branch pushed, no PR |
| `branch-prefix` | no | `publish/` | Branch is `<prefix><name>-<version>` |
| `commit-author-name` | no | `github-actions[bot]` | Git author name |
| `commit-author-email` | no | `41898282+github-actions[bot]@users.noreply.github.com` | Git author email |
| `pr-body` | no | `""` | Extra text appended to the PR body |

Validation always runs with `--allow-warnings` and `--no-overwrite`.

## Outputs

| Output | Description |
|--------|-------------|
| `name` | Pod name read from the podspec |
| `version` | Pod version read from the podspec |
| `branch` | Created branch name |
| `pr-number` / `pr-url` | The opened PR |

## Notes

- **Tags must exist.** Validation builds the pod from `spec.source`, so the tag the podspec points
  at has to be pushed already — which is why the natural trigger is `release: published`.
- **Order matters.** Dependencies are resolved from `sources`, so a pod depending on a new
  `LiveKitWebRTC` / `LiveKitUniFFI` version must be published *after* it.
- **Published specs are immutable.** `--no-overwrite` fails a re-run of an already-merged version
  (`TestPod (0.0.1) already exists and overwriting has been disabled`); bump the version instead.
  Re-running for a version whose PR is still open recreates the branch.

## Testing

`.github/workflows/test.yml` writes a podspec for the two-file Objective-C pod in `test/` —
it validates in seconds — and publishes it to a specs repo of your choosing, at a version derived
from the run number. Point it at a scratch repo, not `livekit/podspecs`.
