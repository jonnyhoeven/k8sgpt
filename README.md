# k8sgpt: The Dev-Ops Bridge

- Mission: To empower every developer—regardless of Kubernetes experience—to identify and resolve kubernetes deployment
  or cluster issues proactively.

Why implement this:

- Knowledge Transfer: Provides human-readable context for complex CrashLoopBackOff or Pending states.
- Daily Health Checks: Automated daily scans via Git ensure our dev clusters don't drift into "unstable" territory.
- Privacy First: Ensure no sensitive logs and infrastructure names (using --anonymize) are exposed to api's

The "Red-to-Green" Dashboard

Instead of complex Grafana dashboards that require manual monitoring, we use a Push Model:

- Detect: Daily Git CI scan using `k8sgpt`.
- Triage: Claude summarizes the impact (Color-coded: Red/Yellow/Green).
- Alert: A concise summary is sent to the team (Slack/Teams).
- Remediate: Use agentic tools to map the error directly back to our YAML manifests for a one-command suggested fix.

![Overview](overview.png)

---

## The Scenario: "A broken and insecure cluster"

We simulate a compromised and unstable Kubernetes cluster hosting critical applications.

| Technical Issue                           | Operational Impact    
|:------------------------------------------|:----------------------|
| **CrashLoopBackOff / ImagePullBackOff**   | **Service Outage:**   |
| **Over-permissive RBAC (Secrets Access)** | **Security Breach:**  |
| **Broken Service Selector**               | **Silent Failure:**   |
| **Privileged Container (NET_ADMIN)**      | **Lateral Movement:** |

---

## Prerequisites

* Kind or Minikube
* kubectl
* k8sgpt CLI
* OpenAI API Key (or a local LLM like Ollama)
* gemini or claude cli

---

## Step-by-Step Guide

### 1. Create the "Vulnerable" Cluster

Create a local cluster:

```bash
brew install kind
kind create cluster --name k8s-rescue
kubectl cluster-info --context kind-k8s-rescue
kubectl get nodes
```

### 2. Injecting "The Chaos"

Apply the manifest in `/manifests/unsafe-app.yaml`. This deployment includes:

- **Privileged Pods:** Containers running as root with `NET_ADMIN` capabilities.
- **RBAC Misconfiguration:** A ServiceAccount that can read all Secrets in the namespace.
- **Plaintext Secrets:** Database passwords exposed in environment variables.
- **Missing Probes:** No liveness/readiness probes, causing "silent" failures.
- **Broken Service:** A service pointing to a non-existent selector.
- **Storage Orphaning:** A PersistentVolumeClaim that can't be fulfilled.

```bash
kubectl apply -f manifests/test-cluster.yaml
```

### 3. Setup k8sgpt

Authenticate with your provider (e.g., OpenAI or Gemini):

```bash
brew tap k8sgpt-ai/k8sgpt
brew install k8sgpt

# OpenAI
k8sgpt auth add --backend openai --model gpt-4
k8sgpt auth default --provider openai
# Gemini
k8sgpt auth add --backend google --model models/gemini-2.5-flash
k8sgpt auth default --provider google
```

### 4. The AI Analysis (Triage Phase)

Run the analysis. By default, it shows the errors. With `--explain`, the LLM kicks in to provide context.

```bash
k8sgpt analyze --explain --filter=Pod,Service,PersistentVolumeClaim,Role,RoleBinding
```

**Demo Point:** Observe how `k8sgpt` translates a `CrashLoopBackOff` or `Pending` state into a human-readable risk
assessment.

### 5. Human Language Querying (Investigation Phase)

Use the filters to ask specific questions about your incident:

```bash
k8sgpt analyze --explain --filter=Ingress
```

### The "Fix-it" Loop (Remediation Phase)

Based on the AI suggestions, apply the hardened configuration:

- **Apply Network Policies:** Restrict traffic between pods.
- **Set Resource Quotas:** Prevent the "Unsafe App" from crashing the node.
- **Fix Selectors:** Align the Service with the actual Pod labels.

### Use the Explain feature:

```bash
k8sgpt analyse --explain
```

### You can drill down with followup questions

Just enable interactive mode

```bash
k8sgpt analyse --explain --interactive
```

Suggested prompts for the demo:

- "Which of these issues is preventing the application from starting?" (Operational focus)
- "Are there any security risks related to secrets?" (Security focus)
- "How do I fix the broken service selector?" (Remediation focus)

### Output the result for other tools or model agents

Generate the report

```bash
k8sgpt analyze --output json | tee  test-cluster-report.json
```

Example: [test-cluster-report.json](test-cluster-report.json)

### Summarized:

We can use [test cluster manifest](manifests/test-cluster.yaml) and [cluster report](test-cluster-report.json) to
summarize/fix most issues in this demo cluster.

Install gemini or claude cli:

```bash
brew install gemini-cli
#Setup login/API-Key
gemini
```

Get a short cluster status report on the `k8sgpt` cluster report and manifests in the test cluster.

```bash
gemini query "@manifests/test-cluster.yaml @test-cluster-report.json
I am providing a k8sgpt report and the associated manifests. Write a short 2 paragraph management readable report.
Reply in the following format:
Color: ||Cluster Color||
Alert: ||Send Alert to ops team (true/false)||
Status: ||Cluster Unicode status||
Report: ||Short report of cluster state||"  | tee test-cluster-status.txt
```

We get concise and short feedback about our cluster and deployments.

```text
Color: Red
Status: 🔴
Report: The cluster is currently in a high-risk state, exhibiting significant security vulnerabilities and configuration
failures. Critical issues include the storage of sensitive credentials in plain text, containers running with elevated
root privileges, and overly permissive RBAC roles that allow unauthorized access to namespace secrets. These
vulnerabilities represent a severe security risk that could lead to data exposure or cluster compromise.

Operational stability is also compromised, with multiple services failing to function correctly. A primary application
service is unreachable due to selector mismatches, and storage provisioning is failing because of references to
non-existent infrastructure components. Immediate intervention is required to rotate exposed secrets, tighten security
policies, and correct manifest configurations to restore service availability and maintain compliance.
Alert: true
```
Example: [test-cluster-status.txt](test-cluster-status.txt)


### Use the interactive chat agent

Query the agent interactively and triage/resolve issues:

```bash
gemini -i "@manifests/test-cluster.yaml @test-cluster-report.json 
I am providing a k8sgpt report and the associated manifests. 
1. Map the errors in the report to the YAML manifests.
2. Triage the issues, prioritizing security vulnerabilities first.
3. Provide actionable insights for each.
4. Ask me which one I want to fix first."
```

### Vibe coding vicious cycle

Please don't use these tools like this:

```bash
gemini -i "@manifests/test-cluster.yaml @test-cluster-report.json 
I am providing a k8sgpt report and the associated manifests. 
Fix all issues!"
```

Keep the dev-ops cycle:

- Fix one issue at a time
- Ask the model of possible solutions/implications
- Request a fix for specific issues
- Review the changes
- Commit
- Merge (4 eyes)
- sync/apply
