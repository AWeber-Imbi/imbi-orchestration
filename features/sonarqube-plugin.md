# Implement SonarQube webhook handling via plugin

https://aweber.ghe.com/apis/sonar-imbi-updater implemented a webhook handler that update Imbi facts based on data retrieve via the
SonarQube API. It used the Imbi v1 API which relying on integer IDs for fact updates and an OpenSearch to find the Imbi project.

I need to implement this as a webhook in imbi-gateway. The filtering and project lookup will be handled by the existing notification
processing mechanism in imbi-gateway. I need to create a SonarQube plugin that retrieves the measurements from SonarQube and updates
the matched imbi project. Look at how webhook processing works. I would like to configure the webhook as:

Third-Party Service: SonarQube
Identifier Selector: /project/key

Rule:

Filter Expression: /branch/is_main==true
Handler: imbi_plugin_sonarqube.actions.update_project_from_webhook
Handler Config:
```json
[
    { "metric": "coverage", "path": "/test_coverage" },
    { "metric": "ncloc", "path": "/lines_of_code" }
]
```

The configuration is a list of mappings from a SonarQube component metric key to a JSONPointer for the PATCH to the imbi project.
The "update_project_from_webhook" action would retrieve:

    https://sonarqube.aweber.io/api/measures/component?component={sonarqube-id}&metricKeys=coverage,ncloc

and then send a PATCH to imbi-api with the path from the configuration and the value from the SonarQube API response. The SonarQube
API endpoint is the `api_endpoint` property on the ThirdPartyService configured for SonarQube. The plugin will be installed onto
this service.

I'm not completely sure where the SonarQube API key will be configured at. Look the imbi-plugin-logzio plugin to see how it
configures the API token.
