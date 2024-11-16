+++
title = "通过 GitLab CI 实现 master 分支到 stg 和 pre 分支的自动同步" 
date = 2024-05-18 
draft = false 
tags = ["GitLab CI/CD", "Development Process Optimization"] 
+++

## 背景

公司的 git flow 要求，在发布代码合并到 `master` 分支后，`stg` 和 `pre` 分支也必须包含最新的 `master` 代码。

<!--more-->

为此，公司设置了一个机器人（bot）来比较 commit 差异。如果在发布代码合并到 `master` 分支一天后，`stg` 和 `pre` 分支仍未更新，团队收到提醒。

然而，我所在的团队成员总是忘记手动执行这个合并操作，导致频繁的提醒，影响了工作效率。

## 解决方案

为了自动化这一流程，避免人为疏忽，我编写了一个 GitLab CI 脚本，一旦 `master` 分支的代码发生变动：

1. **自动合并**：将 `master` 分支的代码自动合并到 `stg` 和 `pre` 分支。
2. **异常处理**：如果合并过程中出现冲突，导致合并失败，脚本会自动创建一个 Merge Request（MR），提醒开发者及时处理。

下面详细介绍该脚本的实现过程。

## 实现步骤

### 1. 添加新的阶段 `merge`

在 GitLab CI/CD 中，`stages` 用于定义流水线中的各个阶段。GitLab Runner 会按照 `stages` 中定义的顺序依次执行任务。

首先，在 `.gitlab-ci.yml` 文件中添加一个新的阶段 `merge`：

```yaml
stages:
  - merge
```

### 2. 添加 `merge-to-stg` 任务

定义一个名为 `merge-to-stg` 的任务，用于将 `master` 分支的代码合并到 `stg` 分支。

```yaml
merge-to-stg:
  stage: merge
  allow_failure: true
  only:
    refs:
      - master
```

**配置说明：**

- `stage: merge`：指定该任务属于 `merge` 阶段，确保其在正确的阶段执行。
- `allow_failure: true`：允许任务失败而不影响整个流水线的状态。即使合并过程中出现冲突，其他任务仍可继续执行，不会被阻塞。
- `only`：
  - `refs: - master`：仅在推送到 `master` 分支时触发该任务，避免在其他分支上执行不必要的操作。

### 3. 编写合并脚本

#### 3.1 配置 Git 用户信息并获取最新代码

在 `script` 中，首先配置 Git 用户信息，并获取远程仓库的最新代码。

```yaml
script:
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_LOGIN"
  - git fetch origin
```

**说明：**

- **配置 Git 用户信息**：使用环境变量 `$GITLAB_USER_EMAIL` 和 `$GITLAB_USER_LOGIN` 设置 Git 用户的邮箱和用户名。
- **获取远程代码**：`git fetch origin` 获取远程仓库的最新提交信息。

#### 3.2 切换到目标分支

切换到需要合并代码的目标分支 `stg`：

```yaml
script:
  - git checkout stg
```

#### 3.3 执行合并并记录状态

执行合并操作，将 `master` 分支的代码合并到当前分支，并推送到远程仓库。然后记录合并状态。

```yaml
script:
  - git merge origin/$CI_COMMIT_REF_NAME
  - git push https://oauth2:$PROJECT_ACCESS_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git
  - echo "SUCCESS" > .job_status
```

**说明：**

- **合并代码**：`git merge origin/$CI_COMMIT_REF_NAME` 将 `origin/master` 分支的代码合并到当前分支。
- **推送代码**：使用项目访问令牌将合并后的代码推送到远程仓库（请确保在项目中设置了 `$PROJECT_ACCESS_TOKEN` 环境变量）。
- **记录状态**：`echo "SUCCESS" > .job_status` 将合并成功的状态记录到 `.job_status` 文件中。

#### 3.4 在任务完成后检查合并状态

使用 `after_script` 定义在任务执行结束后需要执行的命令，检查合并是否成功，并在失败时采取相应措施。

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

**说明：**

- **检查合并状态**：读取 `.job_status` 文件，如果不存在则视为合并失败。
- **根据状态执行操作**：
  - **成功**：输出成功信息。
  - **失败**：输出失败信息，并通过 GitLab API 创建一个 MR，提醒相关人员处理。

### 4. 提取公共脚本片段以减少重复

由于需要对多个分支执行相同的操作，为了提高配置文件的可维护性，我们使用 YAML 的锚点（`&`）和别名（`*`）来提取公共的脚本片段。

#### 4.1 定义公共脚本片段

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

#### 4.2 在任务中引用公共脚本

在刚刚的`merge-to-stg`任务中，修改为使用 `*script_name` 引用预先定义的脚本片段

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

同样，可以这么编写 `merge-to-pre` 任务，将代码合并到 `pre` 分支，避免重复编写相同的代码。

### 5. 完整的 `.gitlab-ci.yml` 文件

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

## 总结

通过以上配置，我们实现了 `master` 分支有任何变更后，自动将其合并到 `stg` 和 `pre` 分支的功能。即使合并过程中出现冲突，脚本也会自动创建 MR，提醒开发者及时处理。这种自动化方式有效地解决了团队成员忘记手动合并的问题，确保了代码在不同环境中的一致性，提高了工作效率。
