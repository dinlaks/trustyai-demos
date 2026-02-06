# Deploying Llama Stack

## Understanding Llama Stack

Llama Stack is an open source standardized, opinionated framework for building and deploying generative AI applications. It provides a unified API interface and building blocks that abstract the complexity of working with large language models, enabling developers to focus on building agentic AI applications rather than managing infrastructure.

As part of Red Hat AI 3 running on this lab, Llama Stack brings enterprise-grade capabilities for deploying production AI workloads with the scalability, security, and operational excellence that Red Hat platforms provide.

### What is Llama Stack?

Llama Stack is designed around the concept of distributions - pre-configured stacks that combine inference engines, model providers, and additional capabilities like retrieval-augmented generation (RAG), safety guardrails, and tool execution. Each distribution is a composition of:

- **Inference providers**: vLLM, AWS Bedrock, Fireworks AI, Together AI, and others
- **Memory and vector stores**: For conversational context and retrieval
- **Safety components**: Content moderation and guardrails
- **Tool runtime**: Integration with external systems and APIs
- **Agent framework**: Orchestration for multi-step reasoning

```
Infrastructure Layer
    ↓
Distribution Layer
    ↓
Llama Stack Unified API Layer
    ↓
Application Layer
```

**Components:**
- OpenShift Platform
- vLLM Instances
- External APIs
- Provider Adapters (vLLM, Bedrock, Fireworks)
- Vector Stores (Chroma, FAISS)
- Safety Runtime (Llama Guard)
- Tool Runtime (Built-in Tools)

**APIs:**
- Inference API
- Agents API
- RAG API
- Safety API
- Memory API
- Tools API

### Why use Llama Stack with Red Hat AI?

Llama Stack on Red Hat OpenShift provides several key advantages for enterprise AI deployments:

1. **Standardization and portability**
   - The unified API allows you to switch between different model providers (vLLM, AWS Bedrock, Azure OpenAI) without rewriting application code
   - Helps address vendor lock-in across hybrid cloud deployments

2. **Enterprise-grade operations**
   - Running on OpenShift provides automated deployment, scaling, monitoring, and security features
   - The Llama Stack operator manages the lifecycle of your AI infrastructure using Kubernetes-native patterns

3. **Production-ready tooling**
   - Agent framework: Multi-turn conversations with tool calling and reasoning
   - RAG support: Vector storage and retrieval for knowledge-grounded responses
   - Safety guardrails: Content moderation and prompt injection protection
   - Observability: OpenAPI-compliant endpoints with health checks and monitoring

4. **Model flexibility**
   - Local inference: Using vLLM for low-latency, cost-effective serving
   - Model-as-a-Service: Integration with cloud providers like AWS Bedrock
   - Mixed deployments: Combine local and remote models based on workload requirements

5. **Developer experience**
   - OpenAPI specification and client SDKs in Python, Node.js, and other languages
   - Consistent developer experience across development, staging, and production environments

### Llama Stack distributions

A distribution defines which providers and capabilities are available in your Llama Stack deployment. The **starter distribution** includes:

- **Inference**: vLLM for local model serving, with optional cloud provider support
- **Vector databases**: For RAG and semantic search
- **Tool runtime**: Built-in tools for web search (Tavily) and RAG
- **Safety**: Content moderation and guardrails
- **Agent orchestration**: Multi-turn conversations with tool calling

This lab uses the starter distribution with vLLM as the primary inference provider, configured to use OpenShift AI's Model-as-a-Service (MaaS) capability.

## Deployment Instructions

### Prerequisites

- Access to an OpenShift cluster with Red Hat AI 3 installed
- OpenShift CLI (`oc`) installed and configured
- Git installed
- User credentials for the OpenShift cluster

### Step 1: Login to OpenShift Cluster

In both Terminal 1 and Terminal 2, login to your OpenShift cluster with the provided credentials:

```bash
oc login <cluster-url> -u <user> -p <password>
```

When prompted about insecure connections, answer `y`:

```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y
```

You should successfully login and see your project `agentic-<user>`:

```
Login successful.

You have one project on this server: "agentic-<user>"

Using project "agentic-<user>".
Welcome! See 'oc help' to get started.
```

### Step 2: Verify LlamaStack CRD Availability

Verify that the LlamaStackDistribution Custom Resource Definition is available:

```bash
oc api-resources | awk 'NR==1 || $1 ~ /llama/ {print $5 "\t" $4}' | column -t
```

Expected output:
```
KIND                    NAMESPACED
LlamaStackOperator
LlamaStackDistribution  true
```

### Step 3: Clone Project Files

Clone the project repository:

```bash
cd $HOME
git clone https://github.com/burrsutter/fantaco-redhat-one-2026
cd fantaco-redhat-one-2026/
```

### Step 4: Create Llama Stack ConfigMap

Apply the Llama Stack configuration:

```bash
oc apply -f llama-stack-scripts/llamastack-configmap.yaml
```

Verify the ConfigMap was created:

```bash
oc get cm
```

You should see `llama-stack-config` in the list:

```
NAME                          DATA   AGE
config-service-cabundle       1      65m
config-trusted-cabundle       1      65m
kube-root-ca.crt              1      65m
llama-stack-config            1      8s
odh-kserve-custom-ca-bundle   1      63m
odh-trusted-ca-bundle         2      64m
openshift-service-ca.crt      1      65m
```

View the ConfigMap details:

```bash
oc get configmap llama-stack-config -o yaml
```

### Step 5: Review Available Models (Optional)

Review the models available on the provided Model as a Service (MaaS):

```bash
curl -sS <litellm_api_base_url>/models \
  -H "Authorization: Bearer <litellm_virtual_key>" | jq
```

Replace `<litellm_api_base_url>` and `<litellm_virtual_key>` with your actual values.

### Step 6: Deploy Llama Stack Distribution

Deploy the Llama Stack Distribution for vLLM using Model-as-a-Service (MaaS):

```bash
oc apply -f - <<'EOF'
apiVersion: llamastack.io/v1alpha1
kind: LlamaStackDistribution
metadata:
  annotations:
    openshift.io/display-name: llamastack-distribution-vllm
  name: llamastack-distribution-vllm
  labels:
    opendatahub.io/dashboard: 'true'
spec:
  replicas: 1
  server:
    containerSpec:
      command:
        - /bin/sh
        - '-c'
        - llama stack run /etc/llama-stack/run.yaml
      env:
        - name: VLLM_TLS_VERIFY
          value: 'false'
        - name: MILVUS_DB_PATH
          value: ~/.llama/milvus.db
        - name: FMS_ORCHESTRATOR_URL
          value: 'http://localhost'
        - name: VLLM_MAX_TOKENS
          value: '4096'
        - name: VLLM_API_URL
          value: '<litellm_api_base_url>'
        - name: VLLM_API_TOKEN
          value: '<litellm_virtual_key>'
        - name: LLAMA_STACK_CONFIG_DIR
          value: /opt/app-root/src/.llama/distributions/rh/
      name: llama-stack
      port: 8321
      resources:
        limits:
          cpu: '2'
          memory: 12Gi
        requests:
          cpu: 250m
          memory: 500Mi
    distribution:
      name: rh-dev
    userConfig:
      configMapName: llama-stack-config
EOF
```

**Important**: Replace the following placeholders in the YAML:
- `<litellm_api_base_url>`: Your LiteLLM API base URL
- `<litellm_virtual_key>`: Your LiteLLM virtual key/token

Expected output:
```
llamastackdistribution.llamastack.io/llamastack-distribution-vllm created
```

### Step 7: Verify Deployment

In Terminal 2, watch the pod status until it's running:

```bash
oc get pods -w
```

Expected output:
```
NAME                                           READY   STATUS    RESTARTS   AGE
llamastack-distribution-vllm-94bf699bf-84sx9   1/1     Running   0          2m59s
```

Press `CTRL+C` to stop watching once the pod is in `Running` state.

### Step 8: Verify Service

In Terminal 1, check that the LlamaStack service is created:

```bash
oc get services
```

Expected output:
```
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
llamastack-distribution-vllm-service   ClusterIP   172.231.210.76   <none>        8321/TCP   3m46s
```

The service is available at port `8321` and can be accessed within the namespace using the service name `llamastack-distribution-vllm-service`.

## Next Steps

After successful deployment, you can:

1. **Access the Llama Stack API**: The service is available at `llamastack-distribution-vllm-service:8321` within your namespace
2. **Test the deployment**: Use the OpenAPI endpoints to interact with the Llama Stack server
3. **Configure applications**: Connect your applications to the Llama Stack service using the unified API

## Troubleshooting

### Pod not starting

If the pod is not starting or stuck in `Pending` state:

```bash
oc describe pod <pod-name>
oc logs <pod-name>
```

### Service not accessible

Verify the service endpoints:

```bash
oc get endpoints llamastack-distribution-vllm-service
```

### Check ConfigMap

Verify the ConfigMap is correctly mounted:

```bash
oc exec <pod-name> -- ls -la /etc/llama-stack/
```

## Additional Resources

- [Llama Stack Documentation](https://docs.llamastack.ai/)
- [Red Hat OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai/)
- [OpenShift CLI Reference](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/)
