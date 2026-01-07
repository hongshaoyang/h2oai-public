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