
# spk infra (under the hood)

A breakdown of how `spk infra` will handle versioning, cloning, and template generation.

## Sourcing Templates (Git Cloning)

`spk` will rely on __git cloning__ your source repository (e.g. microsoft/bedrock) as a means to appropriately source Terraform templates. This will happen as part of the `spk infra scaffold` execution using arguments `--source`, `--version`, and `--template`. The `--source` argument specifies the git url of your source repo, the `--template` argument specifies the path to the template within the git repo, and the `--version` argument specifies the git repo _tag_. `spk` requires that template or repo versions are made in the form or repo tags.

The following sequence of events will take place with regards to sourcing templates when running `spk infra` commands:

1. `spk infra scaffold` will clone source repository to `~/.spk/infra` directory.
2. The `--version` argument will correspond to a repo tag. After the repository is successfully cloned, `git checkout tags/<version>` will checkout the specific version of the repo.
3. `spk infra scaffold` will copy the template files over to the current working directory. The `--template` argument in `spk infra scaffold` will specify the path to the Terraform template in the source repo (e.g. `/cluster/environments/azure-simple`).
4. Argument values and variables parsed from `variables.tf` and `backend.tfvars` will be concatenated and transformed into a `definition.json`.
5. `spk infra generate` will parse the `definition.json` (in the current working directory), and (1) validate the source repo is already cloned in `~./spk/infra` (2) perform a `git pull` to ensure remote updates are merged (3) git checkout the repo tag based on the version provided in the `definition.json` and (4) after it is finished iterating through directories, copy the Terraform template to the `generated` directory with the approprite Terraform files filled out based on `definition.json` files.

## Iterating Through Infrastructure Hierarchy

In a multi-cluster scenario, your infrastructure hierarchy should resemble a tree-like structure as shown:

```
discovery-service
    |- definition.json
    |- east/
        |- definition.json
        |- generated/
            |- main.tf
            |- variables.tf
            |- backend.tfvars
            |- spk.tfvars
    |- west/
        |- definition.json
        |- generated/
            |- main.tf
            |- variables.tf
            |- backend.tfvars
            |- spk.tfvars
```

`spk infra generate` will attempt to recursively read `definition.json` files following a "top-down" approach. When a user executes `spk infra generate east` for example (assuming in `discovery-service` directory):

1. The command recursively (1) reads in the `definition.json` at the current directory level, (2) applies the `definition.json` there to the currently running dictionary for the directory scope, and (3) descends the path step by step.
2. At the final leaf directory, it creates a generated directory and fills the Terraform definition using the source and template at the specified version and with the accumulated variables.
