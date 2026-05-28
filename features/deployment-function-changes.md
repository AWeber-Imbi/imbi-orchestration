# Remove GitHub specifics from deployment functions

The `imbi_gateway.actions.create_release` and `imbi_gateway.actions.add_deployment_event` contain GitHub specific payload details
which we want to avoid. The notification processing should be purely configuration driven. I should have noticed this when the
`handler_config` was unused. This needs to change so that the details about extracting the version number for the Imbi Release, the
title of the Release, and the version number & environment in the Deployment event are configured as extraction patterns in the
`handler_config`.

The configuration for the `create_release` function should use a value like:
```json
{
  "title_selector": "/deployment/description",
  "version_expression": "deployment.ref.matches('^[0-9]+\\.[0-9]+\\.[0-9]+') ? deployment.ref : deployment.sha.substring(0, 7)"
}
```
where `version_expression` is a CEL expression that selects the version number to use in release creation and `title_selector` is a
JSONPointer that selects the title. The user will configure this when they are creating the webhook. This decouples the
implementation from the notification payload body and places the "smarts" about what the payload is in the configuration.

The configuration for the `add_deployment_event` is similar but slightly simpler:
```json
{
  "environment_selector": "/deployment_status/environment",
  "version_expression": "deployment.ref.matches('^[0-9]+\\.[0-9]+\\.[0-9]+') ? deployment.ref : deployment.sha.substring(0, 7)"
}
```
where the `version_expression` is the same CEL expression and the `environment_selector` is a JSONPointer that selects the
environment that is being deployed to.

Examine the imbi-gateway code and update it. Make sure that the tests pass.
