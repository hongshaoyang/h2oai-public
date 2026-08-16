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

models
- [x] test cost constraints
- [x] fix useModelsHub:
  - add global.modelHub.component = modelshub
  - remove references to HF_ENDPOINT in gpt-server, llama 3.1, deepseek R1
  - (optional) set `<VLLM>.podSpec.useModelHub: true` 


telemetry <> gpte integration
```yaml
h2ogpte:
  configmap:
    mux-extra-cm:
      data:
        # enable
        H2OGPTE_MUX_TELEMETRY_ENABLED: "true"
        H2OGPTE_MUX_TELEMETRY_ADDRESS: "tz-testing-prod-telemetry-service.telemetry.svc.cluster.local:80"

  deployment:
    mux:
      replicas: 1
      podSpec:
        containers:
          mux:
            envFrom:
              - configMapRef:
                  name: mux-config
              - configMapRef:
                  name: mux-extra-cm
```


+ the h2ogpte mux service acc need to be allowed in telemetry-server-conf config map
see haic-installer-bundle/terraform/modules/applications/telemetry/resources/telemetry-values.yaml
- check config.authorizedServiceAccounts

### 2025-03-15 vllm addition

```yml
h2ogpt:
  config:
    vllm:
      shaoqwen25vl32b:
        enabled: false
        resources:
          limits:
            memory: 12Gi
            gpu: 1
          requests:
            memory: 4Gi
            cpu: 2000m
            gpu: 1
        storage:
          size: 512Gi
          useEphemeral: true
        pdb:
          enabled: false
        # -- NCCL_IGNORE_DISABLED_P2P
        ncclIgnoreDisabledP2P: "1"
        # -- model to run
        model: "Qwen/Qwen2.5-VL-3B-Instruct"
        # -- update with the LLM model serving
        containerArgs:
          - "--tensor-parallel-size" # 1 for 1 gpu
          - "1"
          - "--gpu-memory-utilization"
          - "0.92"
          - "--dtype"
          - "half" # explicit for mha-testing-v2 setup
          - "--max-model-len"
          - 4K
          - "--seed"
          - "1234"
          - "--trust-remote-code"
        env:
          VLLM_NO_USAGE_STATS: "1"
          DO_NOT_TRACK: "1"
        extraParams:
          max_output_seq_len: 4096
          max_seq_len: 4096
        startupProbe:
          enabled: false
        livenessProbe:
          enabled: false
        readinessProbe:
          enabled: false
```

### prod upgrade

Q: need to fill up appstore.config.server.clientSecret and waveClientSecret?

    Short answer: no need to fill manually, chart handles it end-to-end.

    What they are:
    - clientSecret — secret for Keycloak confidential client hac-appstore (browser session OIDC auth to appstore backend)
    - waveClientSecret — secret for Keycloak confidential client hac-appstore-wave (Wave app/instance OIDC auth)

    Flow for a new install (leave both blank):
    1. values.yaml:1064-1070 — the appstore-secrets K8s Secret template auto-generates a 32-char random string for each if you don't set them, and stores it in Secret <release>-appstore-secrets in the appstore namespace.
    2. templates/keycloak/_secret.tpl:282-308 — the realm import JSON (000-realm.json, consumed by the keycloak-config-cli job) reads that same generated secret via h2o.common.helpers.secret.get and uses it as the secret field when creating the hac-appstore / hac-appstore-wave clients in Keycloak.

    So appstore's own config and the Keycloak client end up with matching secrets automatically — keycloak-config-cli does the work for you.

    Bonus: the random-string helper (h2o.common.helpers.secret.value → h2o.common.libs.randomString) does a lookup on the existing Secret first, so on helm upgrade it won't rotate the value and break existing OIDC sessions — it's stable, not just install-time.

    Only fill these in manually if you want a fixed/known secret (e.g. for GitOps reproducibility or rotating a compromised secret).



