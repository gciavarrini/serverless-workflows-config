Create OCP project
===========

# Configuration
Set `TARGET_NS` to the target namespace:
```console
TARGET_NS=sonataflow-infra
```

# Installation
## Persistence pre-requisites
If persistence is enbaled, you must have a PostgreSQL instance running in the cluster, in the same `namespace` as the workflows.

A `secret` containing the instance credentials must exists as well. 

See https://www.parodos.dev/orchestrator-helm-chart/postgresql on how to install a PostgreSQL instance. Please follow the section detailing how to install using helm. In this document, a `secret` holding the credentials is created.


## Installing helm chart 
From `charts` folder run 
```console
helm install create-ocp-project create-ocp-project --namespace=${TARGET_NS}
```

### Post-installation
#### Persistence

You need to edit the `sonataflow` resource to set the correct value for the `persistence` `spec`.
The defaults are:
```
persistence:
  postgresql:
    secretRef:
      name: sonataflow-psql-postgresql
      userKey: postgres-username
      passwordKey: postgres-password
    serviceRef:
      name: sonataflow-psql-postgresql
      port: 5432
      databaseName: sonataflow
      databaseSchema: create-ocp-project
```

Make sure the above values match what is deployed on your namespace `TARGET_NS`.

You can patch the resource by running (update it if needed with your own values):
```bash
  oc patch sonataflow/create-ocp-project \
    -n ${TARGET_NS} \
    --type merge \
    -p '
    {
      "spec": {
        "persistence": {
          "postgresql": {
            "secretRef": {
              "name": "sonataflow-psql-postgresql",
              "userKey": "postgres-username",
              "passwordKey": "postgres-password"
            },
            "serviceRef": {
              "name": "sonataflow-psql-postgresql",
              "port": 5432,
              "databaseName": "sonataflow",
              "databaseSchema": "create-ocp-project"
            }
          }
        }
      }
    }'
```
#### Environment variables
#### Secret

We also need to set the following environment variables:
* JIRA_API_TOKEN
* OCP_API_SERVER_TOKEN

To do so, edit the secret `${WORKFLOW_NAME}-creds` and set those values and the one of `NOTIFICATIONS_BEARER_TOKEN`:
```
WORKFLOW_NAME=create-ocp-project
oc -n sonataflow-infra patch secret "${WORKFLOW_NAME}-creds" --type merge -p '{
   "data":{
      "NOTIFICATIONS_BEARER_TOKEN":"'$(oc get secrets -n rhdh-operator backstage-backend-auth-secret -o go-template='{{ .data.BACKEND_SECRET  }}')'"
   },
   "stringData":{
      "JIRA_API_TOKEN":"{{ JIRA_API_TOKEN }}",
      "OCP_API_SERVER_TOKEN":"{{ OCP_API_SERVER_TOKEN }}"
   }
}'
```
If you are using Jira cloud, you can generate the `JIRA_API_TOKEN` using https://id.atlassian.com/manage-profile/security/api-tokens 

The `OCP_API_SERVER_TOKEN` should be associated with a service account.

Once the secret is updated, to have it applied, the pod shall be restarted. 
Note that the modification of the secret does not currently restart the pod, the action shall be performed manually or, if you are following the next section, any change to the sonataflow CR will restart the pod.

Note that if you run the `helm upgrade` command, the values of the secret are reseted.

##### Sontaflow CR
Run the following to set the following environment variables values in the workflow:
```console
oc -n sonataflow-infra patch sonataflow create-ocp-project --type merge -p '{
  "spec": {
    "podTemplate": {
      "container": {
        "env": [
          {
            "name": "JIRA_URL",
            "value": "<jira url with schema prefix (http(s)://)>"
          },
          {
            "name": "JIRA_USERNAME",
            "value": "<jira username>"
          },
          {
            "name": "OCP_API_SERVER_URL",
            "value": "<OCP API URL with schema prefix (http(s)://)>"
          },
          {
            "name": "OCP_CONSOLE_URL",
            "value": "<OCP console URL with schema prefix (http(s)://)>"
          }
        ]
      }
    }
  }
}
'
```
