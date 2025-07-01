# 🕒 Kubernetes: Measuring Pod Time to Readiness

This repository provides a Bash script to calculate how long each pod in a Kubernetes Deployment takes to become **Ready** after being created.

## ✅ Use Case
<br>
- Debugging slow pod startups  
- Validating readiness probe configuration  
- Benchmarking deployment behavior across environments  

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

# Default value
DEFAULT_THRESHOLD=30

# Help function
usage() {
  echo -e "Usage: $0 <label-selector> <namespace> [--threshold <seconds>]"
  echo -e "Example: $0 app=myapp dev --threshold 45"
  exit 1
}

# Parse args
while [[ $# -gt 0 ]]; do
  case $1 in
    --threshold)
      THRESHOLD="$2"
      shift 2
      ;;
    -*)
      echo "Unknown option: $1"
      usage
      ;;
    *)
      if [[ -z "$LABEL_SELECTOR" ]]; then
        LABEL_SELECTOR="$1"
      elif [[ -z "$NAMESPACE" ]]; then
        NAMESPACE="$1"
      else
        echo "Too many positional arguments."
        usage
      fi
      shift
      ;;
  esac
done

# Prompt for missing inputs
if [[ -z "$LABEL_SELECTOR" ]]; then
  read -p "Enter label selector (e.g., app=my-app): " LABEL_SELECTOR
fi

if [[ -z "$NAMESPACE" ]]; then
  read -p "Enter namespace: " NAMESPACE
fi

if [[ -z "$THRESHOLD" ]]; then
  read -p "Enter readiness threshold in seconds [default: $DEFAULT_THRESHOLD]: " THRESHOLD
  THRESHOLD=${THRESHOLD:-$DEFAULT_THRESHOLD}  # fallback to default if empty
fi

# Output filename
LABEL_SAFE=$(echo "$LABEL_SELECTOR" | tr '=, ' '_' | tr -dc '[:alnum:]_')
TIMESTAMP=$(date '+%Y%m%d-%H%M%S')
REPORT_FILE="pods_ready_report_${LABEL_SAFE}_${NAMESPACE}_${TIMESTAMP}.csv"

# Color codes
RED='\e[31m'
GREEN='\e[32m'
YELLOW='\e[33m'
RESET='\e[0m'

echo "Pod,Created At,Ready At,Time to Ready (s),Status" > "$REPORT_FILE"
echo -e "${YELLOW}🔍 Checking pods with label selector '${LABEL_SELECTOR}' in namespace '${NAMESPACE}'...${RESET}"

# Get pod list
pods=$(kubectl get pods -n "$NAMESPACE" -l "$LABEL_SELECTOR" -o jsonpath='{.items[*].metadata.name}')

if [[ -z "$pods" ]]; then
  echo -e "${RED}❌ No pods found with label selector '${LABEL_SELECTOR}' in namespace '${NAMESPACE}'.${RESET}"
  exit 1
fi

processed=0

# Process pods
for pod in $pods; do
  creation=$(kubectl get pod "$pod" -n "$NAMESPACE" -o json | jq -r '.metadata.creationTimestamp')
  ready=$(kubectl get pod "$pod" -n "$NAMESPACE" -o json | jq -r '.status.conditions[]? | select(.type=="Ready") | .lastTransitionTime')

  if [[ -z "$creation" || "$creation" == "null" ]]; then
    echo -e "${RED}⚠️  Skipping pod ${pod}: Missing creationTimestamp.${RESET}"
    continue
  fi

  if [[ -z "$ready" || "$ready" == "null" ]]; then
    echo -e "${RED}⚠️  Skipping pod ${pod}: Missing Ready condition transition time.${RESET}"
    continue
  fi

  created_sec=$(date -ud "$creation" +%s)
  ready_sec=$(date -ud "$ready" +%s)
  total_seconds=$((ready_sec - created_sec))
  mins=$((total_seconds / 60))
  secs=$((total_seconds % 60))

  if (( total_seconds > THRESHOLD )); then
    COLOR=$RED
    STATUS="⚠️ Slow Startup"
  else
    COLOR=$GREEN
    STATUS="✅ OK"
  fi

  echo -e "${COLOR}${pod} | Created: $creation | Ready: $ready | Time to Ready: ${total_seconds}s (~${mins}m${secs}s) | $STATUS${RESET}"
  echo "$pod,$creation,$ready,$total_seconds,$STATUS" >> "$REPORT_FILE"
  processed=$((processed + 1))
done

# Final output
if [[ $processed -eq 0 ]]; then
  echo -e "${YELLOW}ℹ️  No valid pods were processed.${RESET}"
else
  echo -e "${GREEN}✅ Report saved to: ${REPORT_FILE} (${processed} pods processed)${RESET}"
fi

```

<br>

## 📊 Sample Outputs

<br>

```
![image](https://github.com/user-attachments/assets/a1236fd0-c0bd-4b25-baf1-8fc9b2731612)
```

<br>

```
![image](https://github.com/user-attachments/assets/ca29abb6-e8a1-4d18-bc37-92773c5f04a6)
![image](https://github.com/user-attachments/assets/1b5e56eb-2c5d-4e80-a6f6-d1f012e94b0c)

```
