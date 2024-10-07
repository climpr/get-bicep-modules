# Get Bicep Modules

This action is used to get a list of normalized, relative module names based on changed `version.json` files or an input module name.

## How to use this action

This action is used by calling it in a step as follows:

### Based on changed files

```yaml
- name: Get Bicep Modules
  uses: climpr/get-bicep-modules@v0
  with:
    root-path: bicep-modules
```

### Based on a single module name

```yaml
- name: Get Bicep Modules
  uses: climpr/get-bicep-modules@v0
  with:
    root-path: bicep-modules
    module-name: name-of-module
```

### Prerequisites

To use this action, the repository must be checked out.

## Parameters

`root-path`: (Required.) The directory in the repo that contains the modules. Example: `bicep-modules`.

`module-name`: (Optional.) The name of the module. This should include the full relative path below the root-path, not including any leading or trailing '/'. Example: 'subnet' or 'modules/subnet'.
Using the module-name parameter will only return the module in the parameter. This can be helpful for workflows with `workflow_dispatch` triggers.

### Examples

#### Used for getting a single module

```yaml
name: Publish Bicep module

on:
  pull_request_target:
    branches:
      - main
    paths:
      - bicep-modules/**/version.json

  workflow_dispatch:
    inputs:
      module-name:
        type: string
        description: "Module name: This should include the full relative tree below the root path. Example: 'subnet' or 'modules/subnet'."
        required: true

env:
  root-path: bicep-modules

jobs:
  publish-bicep-modules:
    runs-on: ubuntu-22.04
    permissions:
      contents: read # Required for repo checkout

    steps:
      - name: Get Bicep Modules
        id: get-module-name
        uses: climpr/get-bicep-modules@v0
        with:
          root-path: bicep-modules
          module-name: ${{ inputs.module-name }}

      - name: Azure login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.APP_ID }}
          tenant-id: ${{ vars.TENANT_ID }}
          subscription-id: ${{ vars.SUBSCRIPTION_ID }}

      - name: publish
        uses: climpr/publish-bicep-module@v0
        with:
          root-path: ${{ env.root-path }}
          module-name: ${{ fromJson(steps.get-module-name.outputs.module-names)[0] }}
          update-parent-versions: true
```

#### Used in a matrix strategy

```yaml
name: Document Bicep modules

on:
  pull_request_target:
    branches:
      - main
    paths:
      - bicep-modules/**/version.json

  workflow_dispatch:
    inputs:
      module-name:
        type: string
        description: "Module name: This should include the full relative tree below the root path. Example: 'subnet' or 'modules/subnet'."
        required: true

      as-pull-request:
        type: boolean
        description: "As Pull Request: Setting this parameter to 'true' will create a pull request instead of committing directly to main."
        required: false
        default: false

env:
  root-path: bicep-modules
  version_psmodules: |
    Az.Accounts:3.0.0
    Az.Resources:7.1.0
    PSDocs.Azure:0.3.0

jobs:
  get-bicep-modules:
    runs-on: ubuntu-22.04
    permissions:
      contents: read # Required for repo checkout
    outputs:
      module-names: ${{ steps.get-module-names.outputs.module-names }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2
          ref: ${{ github.head_ref }}

      - name: Get Bicep modules
        id: get-module-names
        uses: climpr/get-bicep-modules@v0
        with:
          root-path: ${{ env.root-path }}
          module-name: ${{ inputs.module-name }}

  write-readme:
    name: "${{ matrix.module-name }} - Write README"
    if: ${{ needs.get-bicep-modules.outputs.module-names != '' && needs.get-bicep-modules.outputs.module-names != '[]' }}
    runs-on: ubuntu-22.04
    environment: prod
    permissions:
      id-token: write # Required for the OIDC Login
      contents: write # Required for repo checkout and self-commit
      pull-requests: write # Required for creating pull requests
    needs:
      - get-bicep-modules
    strategy:
      matrix:
        module-name: ${{ fromJson(needs.get-bicep-modules.outputs.module-names) }}
      max-parallel: 10
      fail-fast: false

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}

      - name: Azure login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.APP_ID }}
          tenant-id: ${{ vars.TENANT_ID }}
          subscription-id: ${{ vars.SUBSCRIPTION_ID }}

      - name: Write module documentation
        uses: climpr/document-bicep-module@v0
        with:
          root-path: ${{ env.root-path }}
          module-name: ${{ matrix.module-name }}
          as-pull-request: ${{ inputs.as-pull-request }}
```
