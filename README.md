# Checkmarx One++ GitHub Action

This action internally uses the same Checkmarx One CLI as is used by the [Checkmarx AST GitHub Action](https://github.com/marketplace/actions/checkmarx-ast-github-action).  There are a few significant differences with this action:

* The workflow executes both PR decorations with scan summaries and Sarif file uploads to for use with GitHub CodeQL security.
* Dependency resolution for supply chain scanning is performed in the code's build environment for the most accurate results.
* The code's build environment can be located in:
  * A GitHub-hosted runner
  * A self-hosted runner
  * A container

While you will get supply chain scan results using the [Checkmarx AST GitHub Action](https://github.com/marketplace/actions/checkmarx-ast-github-action), the dependency
resolution is performed in a server-side build environment.  The server-side build
environment may not be sufficient for completely accurate supply chain
scan results.

# Quick Start

## Prerequisites

For the Quick Start, the following prerequisites apply:

* You have created an [OAuth client](https://checkmarx.com/resource/documents/en/34965-68612-creating-oauth-clients.html) in Checkmarx One.
* You have defined secrets that are referenced in the workflow.  For example:
  * `${{ secrets.CXONE_TENANT }}` could be set to the name of your Checkmarx One tenant.
  * `${{ secrets.CXONE_CLIENT_ID }}` could be set to the client ID of the created OAuth client.
  * `${{ secrets.CXONE_CLIENT_SECRET }}` could be set to the client secret of the created OAuth client.
* If using a self-hosted runner and a container as the build environment, `docker`
must be installed on the self-hosted runner.

## Example Workflow

If your tenant is in the Checkmarx One US environment, the following example workflow will perform a scan on pull requests or pushes targeting the `main` branch:

```yaml
name: Checkmarx Scan
on:
  workflow_dispatch:
  push:
    branches:
      - main
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
    branches:
      - main
jobs:
  execute-checkmarx-scan:
    permissions:
      security-events: write
      contents: read
      pull-requests: write
      statuses: write
    runs-on: self-hosted

    steps:
    - name: Code Checkout
      uses: actions/checkout@v4
       
    - name: Scan with CxOne++ Action
      id: cxscan
      uses: checkmarx-ts/cxone-plusplus-github-action@v2
      with:
        cx-tenant: ${{ secrets.CXONE_TENANT }}
        cx-client-id: ${{ secrets.CXONE_CLIENT_ID }}
        cx-client-secret: ${{ secrets.CXONE_CLIENT_SECRET }}

    - name: Show Outputs
      shell: bash
      run: |
          echo "Scan ID: ${{ steps.cxscan.outputs.scan-id }}"
          echo "Project ID: ${{ steps.cxscan.outputs.project-id }}"
          echo "Scan Link: ${{ steps.cxscan.outputs.project-deeplink }}"
```

The example has the following properties:

* The tenant is located in the Checkmarx One US multi-tenant environment.
* The scan executes on a self-hosted runner.
* On push to `main` the Sarif results are uploaded to CodeQL security.
* Pull-requests that target the `main` branch will run the action and: 
  * block the PR merge while the scan is executing
  * write a PR comment listing issues discovered during the scan

The additional configuration options can be used to tune the workflow execution.
The `Show Outputs` step in the example is completely optional.

# Additional Configuration Options


## Checkmarx Reference Documentation

This action is an integration that uses the 
[Checkmarx One CLI](https://checkmarx.com/resource/documents/en/34965-68621-checkmarx-one-cli-quick-start-guide.html) and the 
[Checkmarx SCA Resolver](https://checkmarx.com/resource/documents/en/34965-19196-checkmarx-sca-resolver.html).  The inputs to this action translate
to command line options found in the documentation.  Please refer to this
documentation if you need to include additional configuration not explicitly
available through an input.

There are a number of options for both the CxOne CLI and SCA Resolver that can
be provided as environment variables.  Using GitHub's ability to define an
environment for the repository may also be used to inject configuration settings
into the execution of the CxOne CLI and SCA Resolver.


## Inputs

 ### `base-uri`

 The base URI for the region endpoint where your [tenant is located]( https://checkmarx.com/resource/documents/en/34965-68630-configure.html#UUID-494287f8-3019-2542-c111-cb147c286bac_id_ASTCLIConfigurationEnvironmentVariables-CLIConfigurationParameters).

Default: URI for US multi-tenant environment

###  `base-auth-uri`

 The base authentication URI for the region endpoint where your [tenant is located]( https://checkmarx.com/resource/documents/en/34965-68630-configure.html#UUID-494287f8-3019-2542-c111-cb147c286bac_id_ASTCLIConfigurationEnvironmentVariables-CLIConfigurationParameters).

Default: URI for US multi-tenant environment


### `cx-tenant` (required)
**It is advised to store this as a secret value.**

The tenant identifier for your tenant.


### `cx-client-id` (required)
**It is advised to store this as a secret value.**

The OAuth Client ID for API access.

### `cx-client-secret` (required)
**It is advised to store this as a secret value.**

The OAuth client secret key for API access.

### `project-name`

The name of the project where the scan will be executed.

Default: The name of the repository.

### `additional-scan-params`

Additional parameters passed to the CxOne CLI after `scan create`. 

Default: None

### `cx-cli-debug`
If true, turn on debugging for the CxOne CLI and the composite action.

Default: false

### `cx-cli-agent`
The agent name to use when performing CxOne CLI commands.  This value will show as the "Scan Origin" of each scan.

Default: `cxonepp-gh-action`

### `cx-cli-additional-params`

Additional parameters to pass to the CxOne CLI (proxy settings, etc) for all CxOne CLI invocations.

Default: None

### `build-container-tag`

The container tag for the build environment where the scan will execute. If not provided, the scan executes in the runner.



### `docker-login-registry`
The name of the container registry to use for login.

Default: `docker.io`

### `docker-login-username`
**It is advised to store this as a secret value.**

The username for the container registry login.

Default: None
    
### `docker-login-password`
**It is advised to store this as a secret value.**

The password for the container registry login.

Default: None

### `upload-sarif-file`
If true, uploads the Sarif file to create entries on the GitHub security tab
in response to push events handled on the specified branch.

Default: true

### `attach-sarif-file`
If true, attaches the Sarif file to the workflow as an artifact for push or pull request events.

Default: false

### `additional-report-params`
    
Additional parameters used when compiling reports, as described in the CxOne
CLI `results show` command. Do not add the `--filter` option; use the `report-filters` input parameter to set filter options.

### `report-filters`
The criteria that selects what to include in any report results.

Default: `--filter "state=TO_VERIFY;PROPOSED_NOT_EXPLOITABLE;CONFIRMED;URGENT"`

### `attach-sbom-file`
If true, attaches the SBOM file to the workflow as an artifact. 

Default: false

### `sbom-standard`
The SBOM standard format used when generating the SBOM file. 

Default: `CycloneDxJson`

### `show-versions`
If true, emits the versions of the CxOne CLI and SCA Resolver in the action log.

Default: true

## Outputs

### `scan-id`

The GUID identifier for the scan that was performed by the action.

### `project-id`

The GUID identifier for the project where the scan information can be found.

### `project-deeplink`

A deep link to the project overview for the project where the scan was performed.
