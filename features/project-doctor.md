# Project Doctor

The Project Doctor is a tool that analyzes the health and status of a project,
displays it in a project panel similar to the Project Settings. It should use a
little doctor looking icon. Clicking it display the project doctor panel.

## Doctor Panel

* displays the current "Health & compliance" panel with the score breakdown expanded
* includes a utility bar containing the "Recompute Score" and "Sync Deployments"
  actions that are currently in the Project Settings
* displays the results of the last analysis report
* includes a "Analyze" action that triggers a new analysis report

## Analysis Results

* displayed in the Doctor Panel in addition to the health & compliance score breakdown
* each "result" is a panel containing a title, markdown description, and a
  status of "pass", "warn", or "fail"
* each result panel is collapsible and expands to show the markdown description
* results are grouped by severity (pass, warn, fail) and sorted by title
* results are available in Scoring Policies as a new scoring policy type

## Doctor Plugin

We need a new plugin that includes a "analyze" action that implements analysis
logic. The plugin will be associated with the project by adding it to the
project type in the same manner that we do today. I'm not sure how this works
but we have the logzio plugin associated with the API project type somehow. It
can also be associated with any of the Third Party Services that the project has
a `EXISTS_IN` relationship with.

When the Doctor Panel is opened, the "analyze" action is triggered and each of
the plugins runs their associated analysis logic if no analysis report is already
available for the project.

## API Interactions

* the API persists the last analysis report for each project so that it
  can be retrieved quickly when the Doctor Panel is opened
* the API provides an endpoint to trigger a new analysis report
* the API provides an endpoint to retrieve the last analysis report

## UI Interactions

* the Doctor Panel is opened by clicking the doctor looking icon in the project panel
* the icon is colored similar to the health score icon in the project panel but
  based on the severity of the results
