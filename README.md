# Transformations CI/CD Template

### Why
Transformations in CDF are used to process data from RAW to other CDF resource types ("clean"), for example Assets or Time Series.

### How
This template uses Github Workflows to run the `transformations-cli` to deploy transformations to CDF on merges to `main`. If you want to use it with multiple CDF projects, e.g. `customer-dev` and `customer-prod`, you can clone the `deploy-push-main.yml` file and modify it for merges to a specific branch of your choice.

## Repository Layout
In the top-level folder `transformations` you may add each transformation job as a new directory, for example:
```
.
├── .github
│   └── workflows
│       ├── deploy-push-master.yml
│       └── check-pr.yml
├── README.md
└── transformations
    ├── my_transformation_001
    │   ├── manifest.yml
    │   └── transformation.sql
    └── my_transformation_002
        ├── manifest.yml
        └── transformation.sql
```
However, you can pretty much change this layout however you see fit - as long as you obey the following rule: Keep one transformation (manifest+sql) per folder.

### Example layout:
```
.
└── transformations
    ├── assets
    │   └── my_transformation_001
    │       ├── manifest.yml
    │       └── transformation.sql
    ├── time_series
    │   └── my_transformation_002
    │       ├── manifest.yml
    │       └── transformation.sql
    ├── events
    │   └── my_transformation_003
    │       ├── manifest.yml
    │       └── transformation.sql
    ⋮
    └── raw
        └── my_transformation_004
            ├── manifest.yml
            └── transformation.sql
```

## Requirements
### Authentication

```yaml
###########################
# From the workflow file: #
###########################
- name: Deploy transformations
  uses: cognitedata/transformations-cli@main
  env:
      COGNITE_CLIENT_ID: my-cognite-client-id
      COGNITE_CLIENT_SECRET: ...
  with:
      path: transformations
      client-id: <my-transformations-client-id>
      client-secret: ...
      cdf-project-name: <my-project-name>
      token-url: https://login.microsoftonline.com/<my-azure-tenant-id>/oauth2/v2.0/token
      scopes: https://<my-cluster>.cognitedata.com/.default # space separated if multiple

#####################
# In all manifests: #
#####################
authentication:
  # The following are explicit values, not environment variables
  tokenUrl: https://login.microsoftonline.com/<my-azure-tenant-id>/oauth2/v2.0/token
  scopes:
      - https://<my-cluster>.cognitedata.com/.default
  cdfProjectName: <my-project-name>
  # The following are given as the name of an environment variable:
  clientId: ${COGNITE_CLIENT_ID}
  clientSecret: ${COGNITE_CLIENT_SECRET}
```

The one thing to take away from this example is that the manifest variables, like `clientId`, points to the corresponding environment variables specified below `env` in the workflow file. Feel free to name these whatever you want.

By default, we expect you to store the client secrets as secrets in your Github repository. This way we can automatically read them in the workflows, *sweeeeet*. The expected setup is as follows:

```yaml
# Assuming you have one 'main' branch you use for deployments,
# the secrets you need to store are:
TRANSFORMATIONS_CLIENT_SECRET_${BRANCH} -> TRANSFORMATIONS_CLIENT_SECRET_MAIN
COGNITE_CLIENT_SECRET_${BRANCH} -> COGNITE_CLIENT_SECRET_MAIN

# If you need separation of e.g. dev/test/prod, you can then use
# another branch named 'prod' (or test or dev...):
TRANSFORMATIONS_CLIENT_SECRET_${BRANCH} -> TRANSFORMATIONS_CLIENT_SECRET_PROD
COGNITE_CLIENT_SECRET_${BRANCH} -> COGNITE_CLIENT_SECRET_PROD
```

##### Capabilities

You need the following for your *deployment credentials*:
1. Transformations specific capabilities, one of the following is required:
  - Capability: `transformations:read` and `transformations:write`

You need the following for your *read/write credentials*:
1. Capability: `project:list`, `group:list`
2. Required capability to read from source resource type and write to target resource type

### Manifest
The manifest file is a `yaml`-file that describes the transformation, like name, schedule, external ID and what CDF resource type it modifies, - and in what way (like _create_ vs _upsert_)!
#### The required fields are:
```yaml
- externalId
- name
- query        # Relative path to the file containing the SQL query
- destination  # One of: assets, asset_hierarchy, events, timeseries, datapoints, stringdatapoints
```
#### Note on writing to `raw`
When writing to RAW tables, you also need to specify `type`, `rawDatabase` and `rawTable` like this in the yaml-file:
```yaml
destination:
  type: raw
  rawDatabase: someDatabase
  rawTable: someTable
```

#### Required _field_ for auth
__`authentication` must be provided.__

The client credentials to be used in the transformation must be provided with the following syntax:
```yaml
authentication:
  tokenUrl: https://login.microsoftonline.com/<my-azure-tenant-id>/oauth2/v2.0/token
  scopes:
      - https://<my-cluster>.cognitedata.com/.default
  cdfProjectName: <my-project-name>
  clientId: ${COGNITE_CLIENT_ID}          # Env var as referenced in deploy step
  clientSecret: ${COGNITE_CLIENT_SECRET}  # Env var as referenced in deploy step
```

If you want to use separate credentials for read/write, the following syntax is needed:
```yaml
authentication:
  read:
    tokenUrl: https://login.microsoftonline.com/<read-azure-tenant-id>/oauth2/v2.0/token
    scopes:
        - https://<my-cluster>.cognitedata.com/.default
    cdfProjectName: <read-project-name>
    clientId: ${COGNITE_CLIENT_ID}          # Env var as referenced in deploy step
    clientSecret: ${COGNITE_CLIENT_SECRET}  # Env var as referenced in deploy step
  write:
    tokenUrl: https://login.microsoftonline.com/<write-azure-tenant-id>/oauth2/v2.0/token
    scopes:
        - https://<my-cluster>.cognitedata.com/.default
    cdfProjectName: <write-project-name>
    clientId: ${COGNITE_CLIENT_ID_WRITE}          # Env var as referenced in deploy step
    clientSecret: ${COGNITE_CLIENT_SECRET_READ}   # Env var as referenced in deploy step
```


#### The optional fields are:
```yaml
- shared           # Default: true
- schedule         # Default: null (no scheduling!)
- action           # Default: upsert
- notifications    # Default: null
- ignoreNullFields # Default: true
```

#### Valid values for `action`:
1. `upsert`: Create new items, or update existing items if their id or `externalId` already exists.
2. `create`: Create new items. The transformation will fail if there are id or `externalId` conflicts.
3. `update`: Update existing items. The transformation will fail if an id or `externalId` does not exist.
4. `delete`: Delete items by internal id.

#### Schedule
Gotcha: Since __CRON__ expressions typically use a lot of asterisks, (this fellow: `*`), you have to wrap your schedule in single quotes (`'`) if _it starts with one (asterisk)_!
```yaml
schedule: */2 * * * *    # Not valid
schedule: 0/2 * * * *    # Valid, but non-standard CRON
schedule: '*/2 * * * *'  # Valid
```

# Pre-commit
Although this repository has nothing to do with Python, it contains a configuration file for a very neat pip-installable package, `pre-commit`. What it does is install some git-hooks that run automatically when you try to commit changes. The repository also has a workflow that runs on all PRs that uses `yamllint` to verify the `.yaml`-files. The exact command is: `yamllint transformations/`, to avoid checking other yaml-files that might be present in the repo.

What these hooks do, is to run a yaml-formatter, _then yamllint_ to verify. Thus, these tools are only meant for your convenience and can be used by running the follwing from the root of the repository:

```sh
# Assuming you have python available
pip install pre-commit
pre-commit install

# You can also run all files:
pre-commit run --all-files
```

This is only needed the first time, for all future commits from this repo, the hooks will run! :rocket:
