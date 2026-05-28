# Representation of deployments

## After GitHub deployment message

`GET /organizations/aweber/projects/e7YWwJme4WBIBXyADkxjZ/releases/`

```json
[
  {
    "id": "PNKkUOlBsr0wo7lAjOXlC",
    "project_id": "e7YWwJme4WBIBXyADkxjZ",
    "version": "d6bcf577fca77b68df6f845230b8027e393aacdd",
    "title": "Deploy main",
    "description": null,
    "links": [],
    "created_at": "2026-05-08T17:04:31.630270Z",
    "updated_at": "2026-05-08T17:04:31.630270Z",
    "created_by": "admin@example.com"
  }
]
```

## After first GitHub deployment_status message

`GET /organizations/aweber/projects/e7YWwJme4WBIBXyADkxjZ/releases/d6bcf577fca77b68df6f845230b8027e393aacdd/environments`

```json
[
  {
    "environment": {
      "slug": "testing",
      "name": "Testing"
    },
    "deployments": [
      {
        "timestamp": "2026-05-08T17:10:00.423830Z",
        "status": "in_progress",
        "note": null
      }
    ],
    "current_status": "in_progress"
  }
]
```

## After second GitHub deployment_status message

`GET /organizations/aweber/projects/e7YWwJme4WBIBXyADkxjZ/releases/d6bcf577fca77b68df6f845230b8027e393aacdd/environments`

```json
[
  {
    "environment": {
      "slug": "testing",
      "name": "Testing"
    },
    "deployments": [
      {
        "timestamp": "2026-05-08T17:10:00.423830Z",
        "status": "in_progress",
        "note": null
      },
      {
        "timestamp": "2026-05-08T17:16:15.213070Z",
        "status": "success",
        "note": null
      }
    ],
    "current_status": "success"
  }
]
```
