title: Grafana 图表增加 Annotation 标记发布信息
author: tusimo
tags:
  - grafana
  - annotation
  - gitlab
  - ci/cd
categories: []
date: 2024-09-23 14:26:00
---
# 目的
 对于已经绘制好的 `grafana` 的图表,我们在看到一些指标突然异常的时候,往往希望知道这个异常时间点到底发生了什么才导致指标异常的.这个时候排查可能会查看当时是否有功能上线发布. `grafana` 可以对图表增加 `annotation` 标记,天下一些有用的信息记录这个时间点发生的变化.可方便后续进行排查.
 
 如图:
 
![upload successful](/images/pasted-15.png)



![upload successful](/images/pasted-16.png)

# 如何添加

`grafana` 可以通过接口进行添加 `annotation`,这对于自动化构建阶段可以很好的自动添加到 `grafana` 的面板.

文档如下

[annotation  文档](https://grafana.com/docs/grafana/latest/developers/http_api/annotations/#create-annotation)

```
POST /api/annotations HTTP/1.1
Accept: application/json
Content-Type: application/json

{
  "dashboardUID":"jcIIG-07z",
  "panelId":1,
  "time":1507037197339,
  "tags":["tag1","tag2"],
  "text":"Annotation Description"
}

# 其中 dashboardUID 可以不写,不写的话会在所有的图表上增加创建的`annotation`
# panelId 也可以不写,填写会只在这个面板上渲染,其他的不会渲染.
```


## 创建 service account

由于调用后台接口需要认证,需要先创建一个 [`service account` ](https://grafana.com/docs/grafana/latest/administration/service-accounts/)
使用生成的 `token` 调用后台接口进行创建 `annotation`.

创建步骤如下:

![upload successful](/images/pasted-17.png)


![upload successful](/images/pasted-18.png)


![upload successful](/images/pasted-19.png)


![upload successful](/images/pasted-20.png)

保存好上面创建的 `token`,后续调用接口需要使用.

## gitlab cicd 自动创建 Annotation


```
curl -vvv --location 'http://grafana.yourdomain.cn/api/annotations' \
    --header "Authorization: Bearer $GRAFANA_TOKEN" \
    --header 'Content-Type: application/json' \
    --data "{
        \"dashboardUID\":\"$DASHBOARD_ID\",
        \"tags\":[
            \"namespace:$CI_PROJECT_NAMESPACE\",
            \"service:$CI_PROJECT_NAME\",
            \"author:$CI_COMMIT_AUTHOR\",
            \"commit:$CI_COMMIT_TITLE\",
            \"ref:$CI_COMMIT_REF_NAME\",
            \"job:$CI_JOB_ID\",
            \"env:$ENV\"
        ],
        \"text\":\"$CI_COMMIT_AUTHOR: $CI_COMMIT_TITLE\"
    }"

```

以上是使用 `curl` 调用接口自动创建`anotation` 的请求.
其中 `GRAFANA_TOKEN` 是之前创建 `service account` 的 `token`.

该值是通过 gitlab 的 `CI/CD` 的 `Variables` 进行管理的.对于一些敏感的密钥 token 之类的可以通过该功能隐藏,不以明文的形式方式存储在代码里.


![upload successful](/images/pasted-21.png)


同时 `gitlab` 在运行时也会把相关数据存储在运行时的变量里,我们可以获取到例如: 提交代码的 commit信息,提交的内容和作者等等.

我们可以通过如下地址

[CI/CD 预定义变量](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)

```
...
export CI_SERVER_TLS_CA_FILE="/builds/gitlab-examples/ci-debug-trace.tmp/CI_SERVER_TLS_CA_FILE"
if [[ -d "/builds/gitlab-examples/ci-debug-trace/.git" ]]; then
  echo $'\''\x1b[32;1mFetching changes...\x1b[0;m'\''
  $'\''cd'\'' "/builds/gitlab-examples/ci-debug-trace"
  $'\''git'\'' "config" "fetch.recurseSubmodules" "false"
  $'\''rm'\'' "-f" ".git/index.lock"
  $'\''git'\'' "clean" "-ffdx"
  $'\''git'\'' "reset" "--hard"
  $'\''git'\'' "remote" "set-url" "origin" "https://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@example.com/gitlab-examples/ci-debug-trace.git"
  $'\''git'\'' "fetch" "origin" "--prune" "+refs/heads/*:refs/remotes/origin/*" "+refs/tags/*:refs/tags/lds"
++ CI_BUILDS_DIR=/builds
++ export CI_PROJECT_DIR=/builds/gitlab-examples/ci-debug-trace
++ CI_PROJECT_DIR=/builds/gitlab-examples/ci-debug-trace
++ export CI_CONCURRENT_ID=87
++ CI_CONCURRENT_ID=87
++ export CI_CONCURRENT_PROJECT_ID=0
++ CI_CONCURRENT_PROJECT_ID=0
++ export CI_SERVER=yes
++ CI_SERVER=yes
++ mkdir -p /builds/gitlab-examples/ci-debug-trace.tmp
++ echo -n '-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----'
++ export CI_SERVER_TLS_CA_FILE=/builds/gitlab-examples/ci-debug-trace.tmp/CI_SERVER_TLS_CA_FILE
++ CI_SERVER_TLS_CA_FILE=/builds/gitlab-examples/ci-debug-trace.tmp/CI_SERVER_TLS_CA_FILE
++ export CI_PIPELINE_ID=52666
++ CI_PIPELINE_ID=52666
++ export CI_PIPELINE_URL=https://gitlab.com/gitlab-examples/ci-debug-trace/pipelines/52666
++ CI_PIPELINE_URL=https://gitlab.com/gitlab-examples/ci-debug-trace/pipelines/52666
++ export CI_JOB_ID=7046507
++ CI_JOB_ID=7046507
++ export CI_JOB_URL=https://gitlab.com/gitlab-examples/ci-debug-trace/-/jobs/379424655
++ CI_JOB_URL=https://gitlab.com/gitlab-examples/ci-debug-trace/-/jobs/379424655
++ export CI_JOB_TOKEN=[MASKED]
++ CI_JOB_TOKEN=[MASKED]
++ export CI_REGISTRY_USER=gitlab-ci-token
++ CI_REGISTRY_USER=gitlab-ci-token
++ export CI_REGISTRY_PASSWORD=[MASKED]
++ CI_REGISTRY_PASSWORD=[MASKED]
++ export CI_REPOSITORY_URL=https://gitlab-ci-token:[MASKED]@gitlab.com/gitlab-examples/ci-debug-trace.git
++ CI_REPOSITORY_URL=https://gitlab-ci-token:[MASKED]@gitlab.com/gitlab-examples/ci-debug-trace.git
++ export CI_JOB_NAME=debug_trace
++ CI_JOB_NAME=debug_trace
++ export CI_JOB_STAGE=test
++ CI_JOB_STAGE=test
++ export CI_NODE_TOTAL=1
++ CI_NODE_TOTAL=1
++ export CI=true
++ CI=true
++ export GITLAB_CI=true
++ GITLAB_CI=true
++ export CI_SERVER_URL=https://gitlab.com:3000
++ CI_SERVER_URL=https://gitlab.com:3000
++ export CI_SERVER_HOST=gitlab.com
++ CI_SERVER_HOST=gitlab.com
++ export CI_SERVER_PORT=3000
++ CI_SERVER_PORT=3000
++ export CI_SERVER_SHELL_SSH_HOST=gitlab.com
++ CI_SERVER_SHELL_SSH_HOST=gitlab.com
++ export CI_SERVER_SHELL_SSH_PORT=22
++ CI_SERVER_SHELL_SSH_PORT=22
++ export CI_SERVER_PROTOCOL=https
++ CI_SERVER_PROTOCOL=https
++ export CI_SERVER_NAME=GitLab
++ CI_SERVER_NAME=GitLab
++ export GITLAB_FEATURES=audit_events,burndown_charts,code_owners,contribution_analytics,description_diffs,elastic_search,group_bulk_edit,group_burndown_charts,group_webhooks,issuable_default_templates,issue_weights,jenkins_integration,ldap_group_sync,member_lock,merge_request_approvers,multiple_issue_assignees,multiple_ldap_servers,multiple_merge_request_assignees,protected_refs_for_users,push_rules,related_issues,repository_mirrors,repository_size_limit,scoped_issue_board,usage_quotas,wip_limits,adjourned_deletion_for_projects_and_groups,admin_audit_log,auditor_user,batch_comments,blocking_merge_requests,board_assignee_lists,board_milestone_lists,ci_cd_projects,cluster_deployments,code_analytics,code_owner_approval_required,commit_committer_check,cross_project_pipelines,custom_file_templates,custom_file_templates_for_namespace,custom_project_templates,custom_prometheus_metrics,cycle_analytics_for_groups,db_load_balancing,default_project_deletion_protection,dependency_proxy,deploy_board,design_management,email_additional_text,extended_audit_events,external_authorization_service_api_management,feature_flags,file_locks,geo,github_integration,group_allowed_email_domains,group_project_templates,group_saml,issues_analytics,jira_dev_panel_integration,ldap_group_sync_filter,merge_pipelines,merge_request_performance_metrics,merge_trains,metrics_reports,multiple_approval_rules,multiple_group_issue_boards,object_storage,operations_dashboard,packages,productivity_analytics,project_aliases,protected_environments,reject_unsigned_commits,required_ci_templates,scoped_labels,service_desk,smartcard_auth,group_timelogs,type_of_work_analytics,unprotection_restrictions,ci_project_subscriptions,container_scanning,dast,dependency_scanning,epics,group_ip_restriction,incident_management,insights,license_management,personal_access_token_expiration_policy,pod_logs,prometheus_alerts,report_approver_rules,sast,security_dashboard,tracing,web_ide_terminal
++ GITLAB_FEATURES=audit_events,burndown_charts,code_owners,contribution_analytics,description_diffs,elastic_search,group_bulk_edit,group_burndown_charts,group_webhooks,issuable_default_templates,issue_weights,jenkins_integration,ldap_group_sync,member_lock,merge_request_approvers,multiple_issue_assignees,multiple_ldap_servers,multiple_merge_request_assignees,protected_refs_for_users,push_rules,related_issues,repository_mirrors,repository_size_limit,scoped_issue_board,usage_quotas,wip_limits,adjourned_deletion_for_projects_and_groups,admin_audit_log,auditor_user,batch_comments,blocking_merge_requests,board_assignee_lists,board_milestone_lists,ci_cd_projects,cluster_deployments,code_analytics,code_owner_approval_required,commit_committer_check,cross_project_pipelines,custom_file_templates,custom_file_templates_for_namespace,custom_project_templates,custom_prometheus_metrics,cycle_analytics_for_groups,db_load_balancing,default_project_deletion_protection,dependency_proxy,deploy_board,design_management,email_additional_text,extended_audit_events,external_authorization_service_api_management,feature_flags,file_locks,geo,github_integration,group_allowed_email_domains,group_project_templates,group_saml,issues_analytics,jira_dev_panel_integration,ldap_group_sync_filter,merge_pipelines,merge_request_performance_metrics,merge_trains,metrics_reports,multiple_approval_rules,multiple_group_issue_boards,object_storage,operations_dashboard,packages,productivity_analytics,project_aliases,protected_environments,reject_unsigned_commits,required_ci_templates,scoped_labels,service_desk,smartcard_auth,group_timelogs,type_of_work_analytics,unprotection_restrictions,ci_project_subscriptions,cluster_health,container_scanning,dast,dependency_scanning,epics,group_ip_restriction,incident_management,insights,license_management,personal_access_token_expiration_policy,pod_logs,prometheus_alerts,report_approver_rules,sast,security_dashboard,tracing,web_ide_terminal
++ export CI_PROJECT_ID=17893
++ CI_PROJECT_ID=17893
++ export CI_PROJECT_NAME=ci-debug-trace
++ CI_PROJECT_NAME=ci-debug-trace
...

```

## 使用 .gitlab-ci.yml 定义发布

```
stages:
  - deploy
deploy-to-product:
  stage: deploy
  environment: product
  only:
    - master
  script:
    - curl -vvv --location 'http://grafana.yourdomain.cn/api/annotations' \
    --header "Authorization: Bearer $GRAFANA_TOKEN" \
    --header 'Content-Type: application/json' \
    --data "{
        \"dashboardUID\":\"$DASHBOARD_ID\",
        \"tags\":[
            \"namespace:$CI_PROJECT_NAMESPACE\",
            \"service:$CI_PROJECT_NAME\",
            \"author:$CI_COMMIT_AUTHOR\",
            \"commit:$CI_COMMIT_TITLE\",
            \"ref:$CI_COMMIT_REF_NAME\",
            \"job:$CI_JOB_ID\",
            \"env:$ENV\"
        ],
        \"text\":\"$CI_COMMIT_AUTHOR: $CI_COMMIT_TITLE\"
    }"

```

当我们合并代码到`master` 分支后,就会自动打上响应的`annotation` .
这样就可以很方便的定位问题了.






