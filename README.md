# Label classifier

Classifies pull requests by label policy. For every configured label, the
package asks a System One (Jev) evaluation API one independent yes/no question
about one shared pull request context — title, description, commit messages,
changed file paths, diff, and documentation from the base revision — and
combines the returned probabilities with per-label thresholds in PHP.

The command only ever **suggests** labels; the caller decides what to do with
the suggestion. Inside a GitHub Actions run, the suggestions are written to
`$GITHUB_OUTPUT` as `labels=` (suggested for applying) and `advisory=`
(threshold-clearing labels that are reported but not suggested for applying).

## Using it from a GitHub workflow

Define this workflow in the consuming repository. It fires when a pull
request opens: the action suggests labels, applies them by adding labels
only — so existing labels are preserved and nothing is removed — and posts
one comment saying the labels were applied automatically from AI suggestions
and should be checked by a human:

```yaml
name: "Label pull request"

# Runs with secrets against pull requests from forks, so it must never check
# out or execute the pull request head. This action only reads the event
# payload and installs its own pinned code.
on:
  pull_request_target:
    types:
      - "opened"
  workflow_dispatch:
    inputs:
      number:
        description: "Pull request number to label when triggering manually"
        required: true
        type: "string"

permissions:
  contents: "read"
  issues: "write"
  pull-requests: "read"

jobs:
  label:
    name: "Label pull request"
    runs-on: "ubuntu-latest"

    steps:
      - name: "Suggest and apply labels"
        uses: "paulinevos/label-classifier@v1"
        with:
          number: "${{ inputs.number }}"
        env:
          JEV_API_KEY: "${{ secrets.JEV_API_KEY }}"
```

Set `apply: false` to only suggest — the labels are then available as step
outputs `labels` and `advisory`. Pass `model` or `endpoint` to override the
defaults (`jev-1.13`, the Typesafe endpoint). The manual `workflow_dispatch`
trigger exists because `pull_request: opened` runs the workflow from the base
branch: the automatic trigger only goes live once the workflow is merged,
while a manual run with a pull request number works immediately. The same workflow ships as a workflow template in this repository's
`.github/workflow-templates/`.

What the consuming repository needs:

| Requirement | Purpose |
| --- | --- |
| `JEV_API_KEY` secret | authenticates against the evaluation API |
| `issues: write` permission | applying labels |

## Configuration

`config/labels.json` defines one policy per label: the question asked, the
threshold on the yes-probability, and the application mode.

| Application mode | Behavior when the answer clears the threshold |
| --- | --- |
| `auto` | the label is suggested |
| `advisory` | the label is reported separately, never suggested for applying |
| `manual` | the label is never suggested; for labels decided by release management |

## Overrides

Every policy comes from one JSON file, so repositories with their own label
vocabulary pass their own file instead of editing this package:

```bash
bin/label-pull-request --labels-config=/path/to/labels.json
```

The file uses the same shape as `config/labels.json` — one entry per label,
each with a `question`, a `threshold` between 0 and 1, and an `application`
mode. Anything the file defines replaces the default policies entirely.

To keep the shipped policies and adjust only individual thresholds, pass a
comma-separated list; each entry must name a configured label:

```bash
bin/label-pull-request --thresholds="Bug=0.9,dependencies=0.95"
```

The band below a threshold in which a case counts as abstained instead of
rejected works the same way:

```bash
bin/label-pull-request --abstain-margin=0.1
```

Each of these has an environment variable equivalent for workflow use:
`LABEL_THRESHOLDS`, `LABEL_ABSTAIN_MARGIN`, and `LABEL_DOCS_PATHS` (the
documentation paths loaded from the base revision).

## Tests

```bash
composer test
```
