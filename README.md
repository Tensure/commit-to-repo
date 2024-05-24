# Commit To Repo
"Commit To Repo" is a [composite action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) for Github Actions workflows that handles committing file changes to another repository of your choosing. This was originally developed to handle Kubernetes manifest changes for an environment with [ArgoCD](https://argo-cd.readthedocs.io/en/stable/), but can be used in any scenario.

## Steps
This composite action performs the following steps:

1. If additional authentication is provided, the `GITHUB_TOKEN` environment variable is overwritten
1. Repository specified (and branch if provided) is cloned into a sub directory using the `GITHUB_TOKEN` as the authentication mechanism.
1. Files requested are added to the repository in the root or in the path you choose.
1. Files are staged (`git add <file>`), committed, and pushed.

## Inputs

### Required
These inputs are required.

| name | purpose |
| --- | --- |
| github-owner | The owner of the destination repository. This could be an organization or user. |
| github-repository | The destination repository name. |
| files | A multiline string containing the files you want to add changes for. |

### Additional
These inputs are not required, but should be set depending on your scenario.

| name | purpose |
| --- | --- |
| branch | A branch to commit to. Must exist already. |
| path | A path to place all files under |
| github-pat | A personal access token to use for Git authentication. <br/>Use this or `github-app-id` and `github-app-private-key` for committing to repositories other than the one the workflow is executing in. |
| github-app-id | The app ID of a Github App to use for Git authentication. Do not use if `github-pat` is provided.|
| github-app-private-key | The private key of a Github App to use for Git authentication. Do not use if `github-pat` is provided. |

## Usage
Below are some quick usage examples. Please consult [action.yml](./action.yml) for more details.

### Simple
In your Github Actions workflow, add a step referencing this Action. A basic example would look like the one below. The required input values are using variables from the workflow instead of static values.

```yaml
name: cd
on:
  push:
    paths:
      - src/*
jobs:
  delivery:
    runs-on: ubuntu
    steps:
      - name: Generate Some File
        run: echo "some content" >> myfile.txt

      - name: Commit To Repo
        uses: Tensure/commit-to-repo@v1
        with:
          github-owner: ${{ github.event.repository.owner }}
          github-repository: ${{ github.event.repository.name }}
          files: |
            myfile.txt
```

The above example would add the file `myfile.txt` as a new commit to `Tensure/commit-to-repo`. It uses the default runner token for security. This is useful for committing to the repo the workflow is executing in. Ideally, you'd be copying files that are part of some build output. In our use case, the file was a `.yaml` manifest from `helm`.

Be careful to take note that if you have workflows monitoring `push` events with no `path` or `branch` specificiations, you could get into an infinite Action loop because the new commit would retrigger the workflow.

### Specific Branch
If you need to place generated files in a specific branch, use the `branch` input.

```yaml
- name: Commit To Repo
  uses: Tensure/commit-to-repo@v1
  with:
    branch: my-branch
    github-owner: ${{ github.event.repository.owner }}
    github-repository: ${{ github.event.repository.name }}
    files: |
      myfile.txt
```

### Sub Path
If you need to place generated files in a new or existing directory, use the `path` input.

```yaml
- name: Commit To Repo
  uses: Tensure/commit-to-repo@v1
  with:
    github-owner: ${{ github.event.repository.owner }}
    github-repository: ${{ github.event.repository.name }}
    path: new_directory
    files: |
      myfile.txt
```

### Different Repositories
If you need to commit files to a different repository than the one you are in, you'll need to change the `github-owner` and `github-repository` inputs, as well as setting a different `github-pat` since `${{ github.token }}` only has access to the repository it's running in.

You'll need to:
1. [Generate a personal access token](https://docs.github.com/en/enterprise-server@3.9/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) that has write access to the destination repository
1. [Create a secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository) in your workflow repository
1. [Reference the secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#using-secrets-in-a-workflow) it in your workflow

```yaml
- name: Commit To Repo
  uses: Tensure/commit-to-repo@v1
  with:
    github-owner: my-org
    github-repository: my-different-repo
    github-pat: ${{ secrets.MY_CUSTOM_PAT }}
    files: |
      myfile.txt
```

### Github App Instead Of PAT
If you need to commit files to a different repository but do not want to use a personal access token, than the recommend route is to [create a Github App](https://docs.github.com/en/apps/creating-github-apps). This example won't cover how to create the app or permissions required to give it access to your desintation repository, but will show you how to use the inputs required.

Prerequisites to this example require:
1. [Create the app](https://docs.github.com/en/apps/creating-github-apps/writing-code-for-a-github-app/quickstart), granting it the necessary access to your destination repository. You do not need to enable webhooks or change any of the default settings. You simply need to generate a private key once the app is created and installed on your organization.
1. [Create secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository) in your workflow repository for the Github App ID and the Github App Private key.
1. [Reference the secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#using-secrets-in-a-workflow) it in your workflow

```yaml
- name: Commit To Repo
  uses: Tensure/commit-to-repo@v1
  with:
    github-owner: my-org
    github-repository: my-different-repo
    github-app-id: ${{ secrets.ORG_APP_ID }}
    github-app-private-key: ${{ secrets.ORG_APP_PRIVATE_KEY }}
    files: |
      myfile.txt
```

## Help
If you are experiencing errors with this action, see any of the known issues below.

### No Such File Or Directory
Most likely the file you are requesting to be copied is not in the path you've specified. Try adding an `ls -latr` step in your workflow to deduce if that file is actaully present.

### Unable To Clone Repository
Github responds with a 404 (not found) both when the repository doesn't exist AND when you don't have access to it. You'll want to verify that your repository owner and name are correct, then deduce if your credentials provided (`github-pat` or `github-app-id` and `github-app-private-key`) have access to the repository you are trying to clone.

If you are attempting to clone a repository other than the one that this Action is running in and DID NOT provide credentials, please see [inputs](#inputs) for additional inputs needed when doing so. There is also an example for [using a PAT](#different-repositories) or [github app](#github-app-instead-of-pat) if needed.

1. Attempt to access your repository at `https://github.com/<owner>/<repository>`, replacing the `<values>` with what you provided the Action.
1. Try to clone the repository yourself with the credentials provided. If you are using a Github App, you'll need to [generate a token](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app) from the app ID and private key. Then you can use  `git clone https://x-access-token:<token>@github.com/<owner>/<repository>.git` (replacing the `<values>` with your own) to test access.