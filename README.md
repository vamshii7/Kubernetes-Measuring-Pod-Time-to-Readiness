# 🕒 Kubernetes: Measuring Pod Time to Readiness

This repository provides a Bash script to calculate how long each pod in a Kubernetes Deployment takes to become **Ready** after being created.

Ideal for:
<br>
- Debugging slow pod startups  
- Validating readiness probe configuration  
- Benchmarking deployment behavior across environments  

<br>

## ✅ Use Case
<br>

- During a rollout, the App team and DevOps/SRE needed to measure pod readiness times.  
- This script was quickly put together and turned out to be extremely useful across applications and environments.

<br>

## 🛠️ Prerequisites
<br>

- `kubectl` configured to access your cluster  
- `jq` installed on your system  
- Appropriate RBAC permissions to get pod details  

<br>

## 📜 Script: `time-to-ready.sh`
<br>

```bash
#!/bin/bash

DEPLOYMENT_LABEL="your-app-label"   # Replace with your app/deployment label
NAMESPACE="your-namespace"          # Replace with your Kubernetes namespace
THRESHOLD=30                        # Threshold in seconds for slow startup warning
REPORT_FILE="pods_ready_report.csv"

echo "Pod,Created At,Ready At,Time to Ready (s),Status" > "$REPORT_FILE"

pods=$(kubectl get pods -n "$NAMESPACE" -l app="$DEPLOYMENT_LABEL" -o jsonpath='{.items[*].metadata.name}')

for pod in $pods; do
  creation=$(kubectl get pod "$pod" -n "$NAMESPACE" -o json | jq -r '.metadata.creationTimestamp')
  ready=$(kubectl get pod "$pod" -n "$NAMESPACE" -o json | jq -r '.status.conditions[] | select(.type=="Ready") | .lastTransitionTime')

  if [[ $creation != "null" && $ready != "null" ]]; then
    created_sec=$(date -ud "$creation" +%s)
    ready_sec=$(date -ud "$ready" +%s)
    total_seconds=$((ready_sec - created_sec))
    mins=$((total_seconds / 60))
    secs=$((total_seconds % 60))

    # Determine color code
    if (( total_seconds > THRESHOLD )); then
      COLOR="\e[31m" # Red
      STATUS="⚠️ Slow Startup"
    else
      COLOR="\e[32m" # Green
      STATUS="✅ OK"
    fi
    RESET="\e[0m"

    echo -e "${COLOR}${pod} | Created: $creation | Ready: $ready | Time to Ready: ${total_seconds}s (~${mins}m${secs}s) | $STATUS${RESET}"
    echo "$pod,$creation,$ready,$total_seconds,$STATUS" >> "$REPORT_FILE"
  fi
done

```

<br>

## 📊 Sample Output

<br>

```

✅ my-app-7fbd96d8c7-nz8h9 | Created: 2025-06-27T06:12:43Z | Ready: 2025-06-27T06:13:09Z | Time to Ready: 26s (~0m26s) | ✅ OK  
⚠️  my-app-7fbd96d8c7-xj92s | Created: 2025-06-27T06:12:40Z | Ready: 2025-06-27T06:13:20Z | Time to Ready: 40s (~0m40s) | ⚠️ Slow Startup

```
