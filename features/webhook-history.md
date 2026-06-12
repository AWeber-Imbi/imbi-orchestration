# Webhook History & Performance View

The imbi-gateway receives webhook notifications from external services and processes them. Part of the processing
involves storing the notification in the `events` clickhouse table (see: `imbi_gateway.notifications._record_events`).
I want to add a tab in the admin section of the imbi-ui that shows recent webhook notifications and how they were
dispatched. The display should be a summary line for each notification, showing the timestamp, event type, third party
service, project, and disposition. The summary should be sorted by timestamp, with the most recent notifications at the
top. Make it similar to the current Operations Log display with filters for event type, third party service, and project.
The UI should use the existing `/events` API endpoint to fetch and display the event history.
