# Webhook configuration

## GitHub

* an application is not required for webhook processing
* install and activate the `github-enterprise-cloud` plugin

### Deployment webhook

**Identifier Selector** `/repository/id`

**Rules**

| FIELD                 | VALUE                                                 |
| -----                 | -----                                                 |
| **Filter Expression** | `action=="created" && !has(deployment_status.url)`    |
| **Handler**           | `imbi_gateway.actions.create_release`                 |
| **Configuration*      | `{"title_selector":"/deployment/description","version_expression":"deployment.ref.matches('^[0-9]+\\.[0-9]+\\.[0-9]+') ?  deployment.ref : deployment.sha.substring(0, 7)"}` |

| FIELD                 | VALUE                                                 |
| -----                 | -----                                                 |
| **Filter Expression** | `action=="created" && has(deployment_status.url)`     |
| **Handler**           | `imbi_gateway.actions.add_deployment_event`           |
| **Configuration*      | `{"environment_selector":"/deployment_status/environment","status_selector":"/deployment_status/state","version_expression":"deployment.ref.matches('^[0-9]+\\.[0-9]+\\.[0-9]+') ? deployment.ref : deployment.sha.substring(0, 7)"}` |

### Push events

**Identifier Selector** `/repository/id`

**Rules**

| FIELD                 | VALUE                                                          |
| -----                 | -----                                                          |
| **Filter Expression** | `ref=="refs/heads/main"                                        |
| **Handler**           | `imbi_gateway.actions.update_project`                          |
| **Configuration*      | `[{"path":"/last_commit_at","from":"/head_commit/timestamp"}]` |

### Workflow runs

**Identifier Selector** `/repository/id`

**Rules**

| FIELD                 | VALUE                                                          |
| -----                 | -----                                                          |
| **Filter Expression** | `workflow_run.head_branch == "main" && workflow.name == "CI/CD" && action == "completed" && (workflow_run.conclusion == "success" || workflow_run.conclusion == "failure")` |
| **Handler**           | `imbi_gateway.actions.update_project`                          |
| **Configuration*      | `[{"from":"/workflow_run/conclusion","path":"/ci_build_status"},{"from":"/workflow_run/run_started_at","path":"/last_commit_at"}]` |

| FIELD                 | VALUE                                                          |
| -----                 | -----                                                          |
| **Filter Expression** | `workflow_run.head_branch == "main" && workflow.name == "Deploy" && action == "completed" && (workflow_run.conclusion == "success" || workflow_run.conclusion == "failure" || workflow_run.conclusion == "cancelled")` |
| **Handler**           | `imbi_gateway.actions.update_project`                          |
| **Configuration*      | `[{"from":"/workflow_run/conclusion","path":"/ci_deploy_status"}]` |

