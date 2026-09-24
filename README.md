# Quickstart CI Testing

This repo provides the building blocks to address 2 needs:

* the need to run regular sniff testing on quickstarts to make sure they deploy successfully
* the need for development CI testing in a quickstart repository.

The building blocks support deploying to either an OpenShift cluster or a KinD cluster. When
using a KinD cluster you also have the option of building and pushing the images to a local
registry.

## Quickstart development CI testing
This repo addresses the need for development CI testing in a quickstart repository by providing a set of
reusable actions that can be used to implement test workflows in a quickstart repository. This is achieved
by copying over the actions in the `.github/actions directory` to the quickstart repository and then
adding workflows patterned on the ones in this repo under `.github/workflows`. The copying is needed
as the actions cannot be easily referenced from another repository.

## Quickstart sniff testing
This repo addresses the need for regular sniff testing on quickstarts through a set of workflows that use
the common actions in `.github/actions directory` to deploy and test quickstarts from this repository.
These workflows are scheduled to run on a regular basis and report on the health of the associated quickstarts.

## Current status

| Quickstart | Status |
|---|---|
| IT Self-Service Agent | [![Test - IT Self-Service Agent](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-it-self-service-agent.yml/badge.svg)](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-it-self-service-agent.yml) |
| RAG | [![Test - RAG](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-rag.yml/badge.svg)](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-rag.yml) |
| Lemonade Stand Assistant (Default Model) | [![Test - Lemonade Stand Assistant (Default Model)](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-lemonade-stand-assistant-default.yml/badge.svg)](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-lemonade-stand-assistant-default.yml) |
| Speeding Up Ticket Resolution | [![Test - Speeding Up Ticket Resolution](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-speeding-up-ticket-resolution.yml/badge.svg)](https://github.com/rh-ai-quickstart/quickstart-ci-testing/actions/workflows/test-speeding-up-ticket-resolution.yml) |

## Common actions

This repo provides 3 top level actions to be used with test workflows
```
.github/
├── actions/
    ├── setup-and-deploy/       # Prepare runner, clone repo, setup target, print versions, deploy
    ├── run-quickstart-tests/   # Run optional pre-test setup, run tests, upload artifacts
    └── cleanup/                # Tear down deployment (always runs)
```

### Environment variables

These environment variables are used by the actions when appropriate:

| Variable | Used by | Description |
|---|---|---|
| `NAMESPACE` | `setup-and-deploy`, `cleanup` | Kubernetes namespace to deploy into and clean up |
| `REGISTRY` | `setup-and-deploy` | The registry from which to pull images as part of quickstart |

When using the option to build images in KinD, **REGISTRY** is overridden with the address of a local
KinD registry to which the images are built/pushed and then pulled during the deployment.

### setup-and-deploy

Prepares the runner, clones the quickstart repo, logs into OpenShift or creates a KinD cluster, prints environment versions, and deploys the application.

| Input | Required | Default | Description |
|---|---|---|---|
| `repo_url` | yes | — | GitHub URL of the quickstart repo to clone |
| `deploy_command` | yes | — | Command to deploy the application (run from `./quickstart`) |
| `ref` | no | `main` | Branch, tag, or SHA to check out |
| `submodules` | no | `false` | Clone submodules |
| `namespace` | no | `quickstart-ci-project-1` | Kubernetes namespace to deploy into |
| `setup_commands` | no | — | Commands to run after clone but before deploy (e.g. patching chart files) |
| `extra_deploy_args` | no | — | Extra arguments appended to `deploy_command` on all targets |
| `use_kind` | no | `false` | Deploy to a local KinD cluster instead of OpenShift |
| `kind_pre_deploy_command` | no | — | Pre-deploy commands run only on KinD (falls back to `setup_commands` if unset) |
| `kind_extra_deploy_args` | no | — | Extra arguments appended to `deploy_command` only on KinD |
| `build_images` | no | `false` | Build and push images to the local KinD registry before deploying (requires `use_kind: true`) |
| `build_images_command` | no | `make build-all-images` | Command to build images |
| `push_images_command` | no | `make push-all-images` | Command to push images to the local KinD registry |

When deploying to OpenShift, the `PROD_TOKEN`, `PROD_SERVER` environment variables must be passed via `env:` on the action step.
`PROD_SERVER` is the OpenShift cluster to deploy to, and `PROD_TOKEN` is the token for a service account that has access to the
namespace with the privileges required to run the quickstart.

**Namespace:** When copying the common actions to a quickstart repo for workflows running in OpenShift, the default
namespace MUST be changed to a namespace specific to the quickstart otherwise deployments from different
quickstarts may conflict. In addition all workflows that run against the same namespace MUST share a single
concurrency group so only one runs at a time.

### run-quickstart-tests

Runs an optional pre-test setup command, then the test command, and optionally uploads result directories as artifacts.

| Input | Required | Default | Description |
|---|---|---|---|
| `test_command` | yes | — | Command to run the tests (run from `test_working_dir`) |
| `test_working_dir` | no | `./quickstart` | Directory to run test commands in |
| `pre_test_command` | no | — | Setup command to run before the test command |
| `upload_results` | no | — | Space-separated list of directories (relative to `test_working_dir`) to upload as artifacts |
| `upload_retention_days` | no | `2` | Number of days to retain uploaded artifacts |

When using `upload_results` be careful not to include any directories that contain sensitive information as the uploaded GitHub artifacts are public.

### cleanup

Runs the cleanup command to tear down the deployment, then deletes any remaining pods, deployments, services, and PVCs from the namespace. Runs with `continue-on-error` so a failed cleanup never blocks the workflow.

| Input | Required | Default | Description |
|---|---|---|---|
| `cleanup_command` | yes | — | Command to uninstall the application (e.g. `helm uninstall`) |
| `working_dir` | no | `./quickstart` | Directory to run the cleanup command in |

Always use `if: always()` on this step so it runs even when earlier steps fail.

The `cleanup_command` ideally should be a command from the quickstart and MUST remove all artifacts that
have been created by the quickstart, otherwise for OpenShift deployments the namespace may get
polluted and affect future testing.

## Workflow example

An example of a workflow built with the common actions is as shown in the YAML below.
Ideally the quickstart provides common targets like the following which can simply be
called when invoking each of the 3 top level actions to install, test and cleanup:

* make helm-install-test
* make test-short-resp-integration-request-mgr
* make helm-uninstall

If necessary, you can also add additional steps to the workflow like the
`Setup uv` step in the example below, but ideally all of that would
be handled by targets in the quickstart so that the workflow has as little
internal knowledge about the quickstart as possible.

```yaml
name: Test - IT Self-Service Agent

on:
  workflow_dispatch:

concurrency:
  group: quickstart-ci-tests
  cancel-in-progress: false

jobs:
  test:
    name: it-self-service-agent
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: ./.github/actions/setup-and-deploy
        timeout-minutes: 15
        with:
          repo_url: https://github.com/rh-ai-quickstart/it-self-service-agent
          deploy_command: "make helm-install-test"
        env:
          PROD_TOKEN: ${{ secrets.PROD_TOKEN }}
          PROD_SERVER: ${{ secrets.PROD_SERVER }}
          LLM: ${{ vars.LLM_INFERENCE }}
          LLM_ID: ${{ vars.LLM_ID_INFERENCE }}
          LLM_URL: ${{ vars.LLM_URL_INFERENCE }}
          LLM_API_TOKEN: ${{ secrets.LLM_API_TOKEN_INFERENCE }}

      - name: Setup uv
        uses: astral-sh/setup-uv@v6
        with:
          version: "0.8.9"
          python-version: "3.12"

      - uses: ./.github/actions/run-quickstart-tests
        timeout-minutes: 30
        with:
          test_command: "make test-short-resp-integration-request-mgr"
          upload_results: "evaluations/results"
        env:
          LLM: ${{ vars.LLM_INFERENCE }}
          LLM_ID: ${{ vars.LLM_ID_INFERENCE }}
          LLM_URL: ${{ vars.LLM_URL_INFERENCE }}
          LLM_API_TOKEN: ${{ secrets.LLM_API_TOKEN_INFERENCE }}

      - uses: ./.github/actions/cleanup
        if: always()
        with:
          cleanup_command: "make helm-uninstall"

```

## Adding a quickstart to the regular runs

To add a quickstart that will be tested on a regular basis, PR in a workflow to this repository in
`.github/workflows`. By doing so you also commit to check the results on a regular basis
to make sure the tests are passing. It may be easiest to start by copying one of the existing
workflows and then modifying to deploy/test your quickstart.

The PR should also add a status badge for your workflow to the [Current status](#current-status) section above.

As part of the PR approval process you will also need to co-ordinate with the maintainers so that
any additional secrets required are added to the repository (for example to a different model).

## Re-using in a quickstart repository

To use the common actions in your quickstart repository:

* copy the actions in `.github/actions` in this repository into `.github/actions` in your repository
* change the default namespace in `.github/actions/setup-and-deploy/action.yml` to be your new namespace
* If using OpenShift to deploy,
  * create the namespace you specified in the previous step
  * create service account with admin access to that namespace along with any other privileges
    needed to run the quickstart. The agreed convention today is that we don't give these
    service accounts any privileges that would allow changes to be made outside of the namespace
    which was created in the previous step.
  * Add the PROD_TOKEN and PROD_SERVER secrets to your GitHub repo
* create your workflow in `.github/workflows`, ensuring you set the concurrency group to values
   that make sense for your repo
* Add any additional secrets needed by your workflow (for example the LLM_* values in the example)
