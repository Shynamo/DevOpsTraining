# Managing GitLab Workflow

Table of contents:

[TOC]

## Groups

I created a `DevOps` group in GitLab to be able to manage multiple repositories for this project if necessary, and manage labels, CI/CD pipelines and runners at the group level.

To move the `DevOps Training` project into this new group it was as easy as going to `DevOps Training` -> `Settings` -> `General` -> `Advanced` -> `Transfer Project` and transfer it under the `DevOps` group.

As this changed the URL, I had to update my `.git/config`'s URLs to replace `Shynamo` into `devops`:

```toml
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
[remote "origin"]
    url = ssh://git@localhost:2022/devops/devops-training.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
    remote = origin
    merge = refs/heads/main
[lfs]
    repositoryformatversion = 0
    url = "http://localhost:2080/devops/devops-training.git/info/lfs"
    pushurl = "http://localhost:2080/devops/devops-training.git/info/lfs"
[url "http://localhost:2080/"]
    insteadOf = http://localhost/
[lfs "http://localhost:2080/Shynamo/devops/devops-training.git/info/lfs"]
    access = basic
[lfs "http://localhost:2080/devops/devops-training.git/info/lfs"]
    access = basic
```

### Group Labels

I created the following labels in the group namespace:

![gitlab_labels](assets/gitlab_labels.png)

My workflow will be as follows:

There are four **stages** of a task:

Whenever a task comes to mind, I will create a [GitLab Issue](https://docs.gitlab.com/ee/user/project/issues/):

- Issues have the label `Draft` until the are described enough to be sent into `TODO`
- `TODO` issues are ready to develop and could be implemented by anyone knowing the project thanks to their description
- `DOING` are all started issues, even if they are paused
- `DONE` is for issues that have been resolved and merged

`Draft` allows me to quickly create an issue having just a clear title, without describing it yet and break my focus on the current task.

`Draft` and `DONE` could be considered as duplicates of `Open` and `Closed`. However, `DONE` could be used as a verification by team leads before actually closing the issue in an actual development team.

There are three **kinds** of issues:

- `Feature`: A new feature to implement, or task to perform to improve the codebase
- `Bug`: A bug found in the application, to be fixed
- `Report`: A report to write like this one while/after performing a task

Each issue must have at lease a **kind** and a **stage** (`Draft` or `TODO`) when created. [Merge Request](https://docs.gitlab.com/ee/user/project/merge_requests/#merge-requests) should only inherit the **kind** of their parent issue, if any.

Also, I want to automate the label update from `TODO` to `DOING` when a merge request a created from an issue, and from `DOING` to `DONE` when all the merge requests of an issue have been merged.

In my project I will also close the Issues with the former, but in a team the `DONE` tasks should be reviewed by a team leader to know what is going on and to make sure he/she sees it before closing the issue.

#### Issue Templates

First, to make sure labels are set up automatically I use [issue template](https://docs.gitlab.com/ee/user/project/description_templates.html#create-an-issue-template) as well as [quick actions](https://docs.gitlab.com/ee/user/project/quick_actions.html#gitlab-quick-actions).

However, issue template are not yet supported by GitLab, even it their team [is working on it](https://gitlab.com/gitlab-org/gitlab/-/issues/7749). So I must make the templates in my `DevOps Training` project.

To do so, I created [issue templates](../.gitlab/issue_templates) in [.gitlab/issue_templates](../.gitlab/issue_templates) and [merge request templates](../.gitlab/merge_request_templates) in [.gitlab/merge_request_templates](../.gitlab/merge_request_templates) in this repository.

It order to [set a template by default](https://docs.gitlab.com/ee/user/project/description_templates.html#set-a-default-template-for-merge-requests-and-issues), I tried using a symbolic link to `Default.md`:

```cmd
baptiste:~/Projects/GitLab/devops/devops-training/.gitlab/merge_request_templates$ ln -s template.md Default.md ln -s template.md Default.md
```

Spoiler: it does not work.

![gitlab_template_fail](assets/gitlab_template_fail.png)

Only the content of the file is used, which is the path written is the symbolic link.

So, instead of a symbolic link I simply rename `template.md` into `Default.md`, and then it works just fine.

![gitlab_template_success](assets/gitlab_template_success.png)

Unfortunately, the `/unlabel ~Draft ~TODO ~DOING ~DONE` in the merge request template does not work as it seems like issue metadatas are copied after the description's quick actions.

*A **very annoying** problem is that I **cannot click** on an issue or a merge request because of the port redirection. So, I decided to fix this issue right now [using an NGINX a proxy](03-nxginx_proxy.md).*

#### Label Update On Merge Requests

According to ChatGPT, it would be possible doing it using GitLab's API with the following CI/CD pipeline:

```yaml
update_issue_labels:
  script:
    - >
      curl --request PUT
      --header "PRIVATE-TOKEN: $CI_JOB_TOKEN"
      --data "labels=your-labels"
      "$CI_PROJECT_URL/api/v4/projects/$CI_PROJECT_ID/issues/$CI_MERGE_REQUEST_IID"
  only:
    - merge_requests
```

Unfortunately to leads to a right error that may be solved using a different token.

### Merge Request Commit Format

I do not want all the commits of an MR to be shown in the main branch. Additionally, I want to see quickly which Issue or Merge Request is related to a commit, directly in the commit message, such as:

```text
Add GitLab Setup Tutorial (#3 !4)
```

To do so, go to *Settings* -> *Merge requests*.

Here you can:

- Require to squash commit to have only 1 commit per merge request in the main branch
- Require pipelines to succeed and all threads to be resolved
- Update the commit templates

![Merge Request Options](assets/merge_request_options.png)

#### *Merge commit message template*

```text
Merge branch '%{source_branch}' into '%{target_branch}'

%{title}

Closes %{issues}

See merge request %{reference}
```

#### *Squash commit message template*

This will add the Issue and Merge Request links in the commit message, and all the commits as details:

```text
%{title} (%{reference} %{issues})


Commits:
%{all_commits}
```

## CI/CD Pipelines

Fortunately, GitLab provides a few [templates](https://gitlab.com/gitlab-org/gitlab/-/tree/master/lib/gitlab/ci/templates) per programming language, and also a lot of security analyzers.

```YAML
include:
  - template: Code-Quality.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml # Available in GitLab Ultimate
  - template: Security/License-Scanning.gitlab-ci.yml # Available in GitLab Ultimate
  - template: Security/DAST.gitlab-ci.yml # Available in GitLab Ultimate
```

They can also be manually set up using the *Security and Compliance -> Security configuration* panel.

I discovered those pipelines in the well written book *Automating DevOps with GitLab CI/CD Pipelines* written by Christopher Cowell, Nicholas Lotz and Chris Timberlake.
