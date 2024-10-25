+++
title = "Automatically Merging Master Branch Code to Other Branches Using GitLab CI"
date = 2024-05-18
draft = false
tags = ["GitLab CI/CD", "Development Process Optimization"]
+++

## Background

In our development process, it is required that after the release code is merged into the `master` branch, the `stg` and `pre` branches must also contain the latest `master` code.

To ensure this, the company has set up a bot to compare commit differences. If, one day after the release code has been merged into the `master` branch, the `stg` and `pre` branches have not been updated, the team receives a reminder.

However, team members often forget to manually perform this merge operation, leading to frequent reminders and impacting work efficiency.

<!--more-->

## Solution

To automate this process and avoid human oversight, I wrote a GitLab CI script that, once changes occur in the `master` branch:

1. **Automatic Merge**: Automatically merges the code from the `master` branch into the `stg` and `pre` branches.
2. **Exception Handling**: If conflicts occur during the merge process, causing the merge to fail, the script automatically creates a Merge Request (MR) to prompt developers to handle it promptly.

Below is a detailed introduction to the implementation of the script.

## Implementation Steps

### 1. Add a New Stage `merge`

In GitLab CI/CD, `stages` are used to define different stages in a pipeline. GitLab Runner executes tasks in the order defined in `stages`.

First, add a new stage `merge` in the `.gitlab-ci.yml` file:

```yaml
stages:
  - merge
```

### 2. Add the `merge-to-stg` Job

Define a job named `merge-to-stg` to merge the code from the `master` branch into the `stg` branch.

```yaml
merge-to-stg:
  stage: merge
  allow_failure: true
  only:
    refs:
      - master
```

**Configuration Explanation:**

- `stage: merge`: Specifies that this job belongs to the `merge` stage, ensuring it executes in the correct stage.
- `allow_failure: true`: Allows the job to fail without affecting the overall pipeline status. Even if conflicts occur during the merge, other jobs can continue to execute and won't be blocked.
- `only`:
  - `refs: - master`: Triggers this job only when pushing to the `master` branch, avoiding unnecessary operations on other branches.

### 3. Write the Merge Script

#### 3.1 Configure Git User Information and Fetch Latest Code

In the `script` section, first configure the Git user information and fetch the latest code from the remote repository.

```yaml
script:
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_LOGIN"
  - git fetch origin
```

**Explanation:**

- **Configure Git User Information**: Use environment variables `$GITLAB_USER_EMAIL` and `$GITLAB_USER_LOGIN` to set the Git user's email and username.
- **Fetch Remote Code**: `git fetch origin` fetches the latest commit information from the remote repository.

#### 3.2 Switch to the Target Branch

Switch to the target branch where you need to merge the code, `stg`:

```yaml
script:
  - git checkout stg
```

#### 3.3 Perform Merge and Record Status

Execute the merge operation to merge the code from the `master` branch into the current branch, push to the remote repository, and then record the merge status.

```yaml
script:
  - git merge origin/$CI_COMMIT_REF_NAME
  - git push https://oauth2:$PROJECT_ACCESS_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git
  - echo "SUCCESS" > .job_status
```

**Explanation:**

- **Merge Code**: `git merge origin/$CI_COMMIT_REF_NAME` merges the code from the `origin/master` branch into the current branch.
- **Push Code**: Uses the project access token to push the merged code to the remote repository (ensure the `$PROJECT_ACCESS_TOKEN` environment variable is set in the project).
- **Record Status**: `echo "SUCCESS" > .job_status` records the successful merge status into the `.job_status` file.

#### 3.4 Check Merge Status After Job Completion

Use `after_script` to define commands that need to be executed after the job finishes, check whether the merge was successful, and take appropriate actions in case of failure.

```yaml
after_script:
  - job_status=$(cat .job_status 2>/dev/null || echo "FAILED")
  - current_branch=$(git rev-parse --abbrev-ref HEAD)
  - |
    if [ "$job_status" == 'SUCCESS' ]; then
      echo "Merge to $current_branch succeeded."
    else
      echo "Merge to $current_branch failed."
      curl --request POST --header "PRIVATE-TOKEN:$PROJECT_ACCESS_TOKEN" \
      --data "source_branch=$CI_COMMIT_REF_NAME&target_branch=$current_branch&title=Auto merge $CI_COMMIT_REF_NAME to $current_branch" \
      "https://$CI_SERVER_HOST/api/v4/projects/$CI_PROJECT_ID/merge_requests"
    fi
```

**Explanation:**

- **Check Merge Status**: Reads the `.job_status` file; if it does not exist, it is considered a merge failure.
- **Perform Actions Based on Status**:
  - **Success**: Outputs a success message.
  - **Failure**: Outputs a failure message and uses the GitLab API to create an MR, prompting relevant personnel to handle it.

### 4. Extract Common Script Fragments to Reduce Duplication

Since the same operations need to be performed on multiple branches, to improve the maintainability of the configuration file, we use YAML anchors (`&`) and aliases (`*`) to extract common script fragments.

#### 4.1 Define Common Script Fragments

```yaml
.fetch_script: &fetch_script
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_LOGIN"
  - git fetch origin

.merge_script: &merge_script
  - git merge origin/$CI_COMMIT_REF_NAME
  - git push https://oauth2:$PROJECT_ACCESS_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git
  - echo "SUCCESS" > .job_status

.check_status_script: &check_status_script
  - job_status=$(cat .job_status 2>/dev/null || echo "FAILED")
  - current_branch=$(git rev-parse --abbrev-ref HEAD)
  - |
    if [ "$job_status" == 'SUCCESS' ]; then
      echo "Merge to $current_branch succeeded."
    else
      echo "Merge to $current_branch failed."
      curl --request POST --header "PRIVATE-TOKEN:$PROJECT_ACCESS_TOKEN" \
      --data "source_branch=$CI_COMMIT_REF_NAME&target_branch=$current_branch&title=Auto merge $CI_COMMIT_REF_NAME to $current_branch" \
      "https://$CI_SERVER_HOST/api/v4/projects/$CI_PROJECT_ID/merge_requests"
    fi
```

#### 4.2 Reference Common Scripts in Jobs

In the previously defined `merge-to-stg` job, modify it to reference the predefined script fragments using `*script_name`:

```yaml
merge-to-stg:
  stage: merge
  allow_failure: true
  script:
    - *fetch_script
    - git checkout stg
    - *merge_script
  after_script:
    - *check_status_script
  only:
    refs:
      - master
```

Similarly, you can write the `merge-to-pre` job to merge code into the `pre` branch, avoiding duplicate code.

### 5. Complete `.gitlab-ci.yml` File

```yaml
image: your-docker-image

stages:
  - merge

.fetch_script: &fetch_script
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_LOGIN"
  - git fetch origin

.merge_script: &merge_script
  - git merge origin/$CI_COMMIT_REF_NAME
  - git push https://oauth2:$PROJECT_ACCESS_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git
  - echo "SUCCESS" > .job_status

.check_status_script: &check_status_script
  - job_status=$(cat .job_status 2>/dev/null || echo "FAILED")
  - current_branch=$(git rev-parse --abbrev-ref HEAD)
  - |
    if [ "$job_status" == 'SUCCESS' ]; then
      echo "Merge to $current_branch succeeded."
    else
      echo "Merge to $current_branch failed."
      curl --request POST --header "PRIVATE-TOKEN:$PROJECT_ACCESS_TOKEN" \
      --data "source_branch=$CI_COMMIT_REF_NAME&target_branch=$current_branch&title=Auto merge $CI_COMMIT_REF_NAME to $current_branch" \
      "https://$CI_SERVER_HOST/api/v4/projects/$CI_PROJECT_ID/merge_requests"
    fi

merge-to-stg:
  stage: merge
  allow_failure: true
  script:
    - *fetch_script
    - git checkout stg
    - *merge_script
  after_script:
    - *check_status_script
  only:
    refs:
      - master

merge-to-pre:
  stage: merge
  allow_failure: true
  script:
    - *fetch_script
    - git checkout pre
    - *merge_script
  after_script:
    - *check_status_script
  only:
    refs:
      - master
```

## Summary

Through the above configuration, we have implemented the functionality of automatically merging the code from the `master` branch into the `stg` and `pre` branches after code has any changes in `master`. Even if conflicts occur during the merge process, the script automatically creates an MR to prompt developers to handle it promptly. This automation effectively solves the problem of team members forgetting to manually merge, ensures consistency of code across different environments, and improves work efficiency.
