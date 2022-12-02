# Challenge 3: Trigger GitHub Actions with trunk-based development

Trunk-based development means that direct pushes to the main branch is prohibited. Code is to be created/modified solely on feature branches.

## Create a branch protection rule

1. In the GitHub repository, go to the **Settings** tab.
2. Under **Code and automation**, select **Branches**.
3. Choose to **Add rule**.
4. For Branch name pattern, enter `main`.
5. Check the checkboxes for **Require a pull request before merging** and **Require approvals**.
6. Select **Create**.

## Create Echo workflow that triggers on push to main

Create a new file `03-placeholder.yml` in the **.github\workflows** folder.

```yml
name: Challenge 3

on:
  push:
    branches:
      - main

jobs:
  dummyOutput:
    runs-on: ubuntu-latest
    steps:
      - name: Placeholder
        run: |
          echo "Will add code checks here in the next challenge"
```

## Create branch for new code, issue pull request, merge pull request

The person who submits the pull request can't be set to a reviewer, so for demo purposes use the override feature. Check the box next to "Merge without waiting for requirements to be met (bypass branch protections)" and merge the PR.

## Go to Actions tab and watch "placeholder" Challenge 3 action run

The **dummyOutput** step should show the echo.
