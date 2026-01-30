# h2oai-public


### 2025-01-06 mha work session notes
- vex is a thin wrapper over hnswllib - which is an in-memory database. so memory usage is going to increase as the index grows
- ?? what is the replication count on vllm? if there is single pod (or less) then we can focus on the following time stamps for RAG chat and normal chat both
    - time when the query submitted from UI
    - reached the vllm pods
    - response started by vllm ... response end time.
    - response showing up on UI (initial stream) ... and response end time.
- then since there are only 4 vllm pods enable realtime log on all of them (multiple shetll sessions) and try to do a normal chat query. This needs to be done at a time when there are no users in the system so you can assure the logs in vllm pod are result of your UI based chat. note the timestamps as discussed above.
```sh
curl -X POST http://h2oai-h2ogpt-llama-3-1-70b:5000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
    "messages": [
      {"role": "user", "content": "Say hello world"}
    ],
    "max_tokens": 50,
    "temperature": 0.7
  }'
```
- h2ogpt.config.h2ogpt.agent.enabled - remove, take the default (true)
    - remove h2ogpt.config.agent - remove
    - 
- agent waits for inferencing service to be ready
    - gpt-agent deployment init container
    - cm gpt-agent-config-envs, H2OGPT_MODEL_LOCK
    - 
- agentic tools
  - defaultAgentTools: '["shell", "python", "ask_question_about_documents.py", "mermaid_renderer.py", "convert_document_to_text.py", "rag_text"]'

- VLLM tuning
  - --distributed-executor-backend - try using mp if ray doesn't work - https://docs.vllm.ai/en/stable/serving/parallelism_scaling/#single-node-deployment
  - max-model-len - may need to set if needed
  - 

### 2025-01-28 cj for deployment restart

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-restarter
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-restarter-role
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "patch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-restarter-binding
subjects:
  - kind: ServiceAccount
    name: deployment-restarter
roleRef:
  kind: Role
  name: deployment-restarter-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-deployment-restart
spec:
  schedule: "0 3 * * *" # Runs every day at 3:00 AM
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: deployment-restarter
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - kubectl rollout restart deployment/YOUR_DEPLOYMENT_NAME
          restartPolicy: OnFailure
```
script for psql login
```sh
#!/usr/bin/env bash
set -e

ROLE="$1"
if [[ -z "$ROLE" ]]; then
  echo "Usage: $0 {superuser|gpte}"
  exit 1
fi

# Credentials
export superuser='u4eHNiHrGD8oHNMfHWtCzA9ts2l1Fs5u9bMy6mhPVnu61QAq90MGr0Orl77QsZhh'
export h2ogpte='pGwlBimEMnOiiBpWkCDugnxJgoM4zFKGOOlbxDJ2He3yBU9W6Y6lqO2IdShg5pJY'
export readonly='readonly'
export PGSSLMODE=require
export PGPASSWORD="${!ROLE}"

# Start port-forward in background
kubectl port-forward service/haic-db-test-pooler 5432:5432 -n db >/tmp/pg-portforward.log 2>&1 &
PF_PID=$!
trap "kill $PF_PID" EXIT
sleep 2

psql -U "$ROLE" -h localhost -p 5432 -d appstore
```

readonly user
```sql
/**
\l list dbs
\dn - list schemas
\d - list tables/views etc
\dt - list tables


\du - view users
*/

-- db level
SELECT
  datname,
  has_database_privilege('readonly', datname, 'CONNECT') AS connect,
  has_database_privilege('readonly', datname, 'CREATE')  AS create,
  has_database_privilege('readonly', datname, 'TEMP')    AS temp
FROM pg_database
ORDER BY datname;

-- schema level
SELECT
  n.nspname AS schema,
  has_schema_privilege('readonly', n.nspname, 'USAGE')  AS usage,
  has_schema_privilege('readonly', n.nspname, 'CREATE') AS create
FROM pg_namespace n
ORDER BY n.nspname;


CREATE USER readonly WITH PASSWORD 'readonly' -- note, not role

-- db level, run for each db
\c your_db
-- GRANT CONNECT ON DATABASE h2ogpte TO readonly; -- not needed because users have default connect privilege

-- schema level, run for each schema
GRANT USAGE ON SCHEMA h2ogpte TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA h2ogpte TO readonly;



-- verification
select * from h2ogpte.default_prompt_templates limit 1; -- prefix for schema
select * from telemetry limit 1; 
select * from app_name limit 1; -- appstore
```