# GitHub Webhook: Deployments

The imbi-gateway receives GitHub notifications related to `deployment` and `deployment_status` events and uses the Imbi API to
create release objects and update their status based on the webhook content.

## Deployment notifications

**Order of events**

1. `deployment`: action=created
2. `deployment_status`: state=in_progress
3. `deployment_status`: state=success|failure

**Data notes**
- the version for the Imbi Release is included in each message payload at `/deployment/ref`
- each message should be recorded as a separate deployment event on the DEPLOYED_TO edge
- added `/deployment/url` as a note
- the GitHub plugin (imbi-plugin-github subdirectory) is installed into the ThirdPartyService associated with the Webhook
- the GitHub id of user that created the deployment is available as `/deployment/creator/id`

I think that we want to create a new imbi-api endpoint that returns a User from a plugin ID and subject. The GitHub user ID is the
subject that imbi-plugin-github stores in the relationship.

## Scope

- imbi-plugin-github: installed into the imbi instance as a global plugin
- imbi-gateway: implement `actions.create_release` and `actions.add_deployment_event`
- imbi-api: add supporting APIs as necessary

## actions.create_release()

Used as a `WebhookRule.handler` to process `deployment` notifications. The notification payload is described by
https://docs.github.com/en/enterprise-cloud@latest/webhooks/webhook-events-and-payloads#deployment

It should extract the interesting parts of the event using a pydantic model and ensure that the Imbi Release exists for the version
using the "Create Relase" Imbi API:

`POST /api/organizations/{org_slug}/projects/{project_id}/releases/`

## actions.add_deployment_event()

Used as a `WebhookRule.handler` to process `deployment_status` notifications. The notification payload is described by
https://docs.github.com/en/enterprise-cloud@latest/webhooks/webhook-events-and-payloads#deployment_status

It should extract the interesting parts of the event using a pydantic model and  append a new deployment event using the "Record
Deployment" Imbi API:

`POST /api/organizations/{org_slug}/projects/{project_id}/releases/{version}/environments/{env_slug}` 

