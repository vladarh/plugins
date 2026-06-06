Yes. Best way: **export with kubectl from your admin machine**, create archive, then write the archive into Jira pod `/tmp`.

Do **not** try to run `kubectl` inside the Jira pod — usually Jira container has no `kubectl`, no kubeconfig, and no RBAC.

---

## Option A — export locally, save archive into Jira pod `/tmp`

Set variables:

```bash
SRC_NS=C1_NAMESPACE_NAME
TARGET_NS=tech-confluence-jira

JIRA_POD=$(kubectl get pod -n "${TARGET_NS}" --no-headers | awk '/jira/ && $3=="Running" {print $1; exit}')

echo "${JIRA_POD}"
```

Example if C1 namespace is also `tech-confluence-jira`:

```bash
SRC_NS=tech-confluence-jira
TARGET_NS=tech-confluence-jira

JIRA_POD=$(kubectl get pod -n "${TARGET_NS}" --no-headers | awk '/jira/ && $3=="Running" {print $1; exit}')

echo "${JIRA_POD}"
```

---

## Full export script and upload to Jira `/tmp`

This exports:

- all namespaced resources
- Applications / Alauda CRs
- ConfigMaps
- Secrets — full, sensitive
- Pods
- logs
- events
- PVCs
- related PVs
- Istio/Gateway resources

Then uploads archive into:

```text
/tmp/c1-export/
```

inside Jira pod.

```bash
SRC_NS=C1_NAMESPACE_NAME
TARGET_NS=tech-confluence-jira

JIRA_POD=$(kubectl get pod -n "${TARGET_NS}" --no-headers | awk '/jira/ && $3=="Running" {print $1; exit}')

OUT=export-c1-${SRC_NS}-$(date +%Y%m%d-%H%M%S)

mkdir -p "${OUT}"/{resources,applications,configmaps,secrets,pods,logs,describe,events,pv,istio,helm}

echo "Target Jira pod: ${JIRA_POD}"
echo "Export folder: ${OUT}"

kubectl get namespace "${SRC_NS}" -o yaml > "${OUT}/namespace.yaml"

echo "Export all namespaced resources"
kubectl api-resources --verbs=list --namespaced -o name | sort | while read -r RESOURCE; do
  SAFE_NAME=$(echo "${RESOURCE}" | sed 's#[/.]#_#g')
  echo "Exporting ${RESOURCE}"
  kubectl get "${RESOURCE}" -n "${SRC_NS}" -o yaml --ignore-not-found > "${OUT}/resources/${SAFE_NAME}.yaml" 2>"${OUT}/resources/${SAFE_NAME}.err" || true
done

echo "Export Alauda/Application resources"
kubectl api-resources --verbs=list --namespaced -o name | grep -i application | while read -r RESOURCE; do
  SAFE_NAME=$(echo "${RESOURCE}" | sed 's#[/.]#_#g')
  kubectl get "${RESOURCE}" -n "${SRC_NS}" -o yaml --ignore-not-found > "${OUT}/applications/${SAFE_NAME}.yaml" 2>"${OUT}/applications/${SAFE_NAME}.err" || true
done

echo "Export ConfigMaps"
kubectl get configmaps -n "${SRC_NS}" -o yaml > "${OUT}/configmaps/all-configmaps.yaml" 2>/dev/null || true

for CM in $(kubectl get configmaps -n "${SRC_NS}" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null); do
  kubectl get configmap "${CM}" -n "${SRC_NS}" -o yaml > "${OUT}/configmaps/${CM}.yaml" 2>/dev/null || true
done

echo "Export Secrets metadata redacted"
if command -v jq >/dev/null 2>&1; then
  kubectl get secrets -n "${SRC_NS}" -o json \
    | jq 'del(.items[].data, .items[].stringData)' \
    > "${OUT}/secrets/secrets-metadata-redacted.json" 2>/dev/null || true
else
  kubectl get secrets -n "${SRC_NS}" \
    -o custom-columns=NAME:.metadata.name,TYPE:.type,AGE:.metadata.creationTimestamp \
    > "${OUT}/secrets/secrets-list.txt" 2>/dev/null || true
fi

echo "Export full Secrets - SENSITIVE"
kubectl get secrets -n "${SRC_NS}" -o yaml > "${OUT}/secrets/_SENSITIVE_all-secrets-full.yaml" 2>/dev/null || true

echo "Export Helm metadata"
kubectl get secrets -n "${SRC_NS}" -l owner=helm -o yaml > "${OUT}/helm/helm-release-secrets.yaml" 2>/dev/null || true
kubectl get configmaps -n "${SRC_NS}" -l OWNER=TILLER -o yaml > "${OUT}/helm/tiller-configmaps.yaml" 2>/dev/null || true

if command -v helm >/dev/null 2>&1; then
  helm list -n "${SRC_NS}" > "${OUT}/helm/helm-list.txt" 2>/dev/null || true
fi

echo "Export pods"
kubectl get pods -n "${SRC_NS}" -o wide > "${OUT}/pods/pods-wide.txt" 2>/dev/null || true
kubectl get pods -n "${SRC_NS}" -o yaml > "${OUT}/pods/pods.yaml" 2>/dev/null || true

echo "Export describe output"
kubectl describe pods -n "${SRC_NS}" > "${OUT}/describe/pods.txt" 2>/dev/null || true
kubectl describe deployments -n "${SRC_NS}" > "${OUT}/describe/deployments.txt" 2>/dev/null || true
kubectl describe statefulsets -n "${SRC_NS}" > "${OUT}/describe/statefulsets.txt" 2>/dev/null || true
kubectl describe services -n "${SRC_NS}" > "${OUT}/describe/services.txt" 2>/dev/null || true
kubectl describe pvc -n "${SRC_NS}" > "${OUT}/describe/pvc.txt" 2>/dev/null || true
kubectl describe ingress -n "${SRC_NS}" > "${OUT}/describe/ingress.txt" 2>/dev/null || true
kubectl describe gateway -n "${SRC_NS}" > "${OUT}/describe/gateway.txt" 2>/dev/null || true
kubectl describe httproute -n "${SRC_NS}" > "${OUT}/describe/httproute.txt" 2>/dev/null || true

echo "Export logs"
for POD in $(kubectl get pods -n "${SRC_NS}" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null); do
  kubectl logs "${POD}" -n "${SRC_NS}" --all-containers=true --tail=-1 \
    > "${OUT}/logs/${POD}.log" 2>"${OUT}/logs/${POD}.err" || true

  kubectl logs "${POD}" -n "${SRC_NS}" --all-containers=true --previous --tail=-1 \
    > "${OUT}/logs/${POD}.previous.log" 2>"${OUT}/logs/${POD}.previous.err" || true
done

echo "Export events"
kubectl get events -n "${SRC_NS}" --sort-by='.lastTimestamp' > "${OUT}/events/events.txt" 2>/dev/null || true
kubectl get events -n "${SRC_NS}" -o yaml > "${OUT}/events/events.yaml" 2>/dev/null || true

echo "Export PVC/PV"
kubectl get pvc -n "${SRC_NS}" -o wide > "${OUT}/pv/pvc-wide.txt" 2>/dev/null || true
kubectl get pvc -n "${SRC_NS}" -o yaml > "${OUT}/pv/pvc.yaml" 2>/dev/null || true

for PV in $(kubectl get pvc -n "${SRC_NS}" -o jsonpath='{range .items[*]}{.spec.volumeName}{"\n"}{end}' 2>/dev/null); do
  if [ -n "${PV}" ]; then
    kubectl get pv "${PV}" -o yaml > "${OUT}/pv/${PV}.yaml" 2>/dev/null || true
  fi
done

echo "Export Istio/Gateway resources"
for RESOURCE in gateway httproute tcproute tlsroute virtualservice destinationrule serviceentry authorizationpolicy peerauthentication waypoint; do
  kubectl get "${RESOURCE}" -n "${SRC_NS}" -o yaml --ignore-not-found > "${OUT}/istio/${RESOURCE}.yaml" 2>"${OUT}/istio/${RESOURCE}.err" || true
done

echo "Create archive"
tar -czf "${OUT}.tar.gz" "${OUT}"

echo "Create target dir inside Jira pod"
kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "mkdir -p /tmp/c1-export"

echo "Upload archive to Jira pod /tmp/c1-export"
kubectl exec -i -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "cat > /tmp/c1-export/${OUT}.tar.gz" < "${OUT}.tar.gz"

echo "Verify file in Jira pod"
kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "ls -lh /tmp/c1-export/${OUT}.tar.gz"

echo "Done"
echo "Local archive: ${OUT}.tar.gz"
echo "Pod path: /tmp/c1-export/${OUT}.tar.gz"
```

---

## If you want to upload already existing archive

If you already created:

```text
export-c1-tech-confluence-jira-20260606-120000.tar.gz
```

then:

```bash
TARGET_NS=tech-confluence-jira
JIRA_POD=$(kubectl get pod -n "${TARGET_NS}" --no-headers | awk '/jira/ && $3=="Running" {print $1; exit}')

ARCHIVE=export-c1-tech-confluence-jira-20260606-120000.tar.gz

kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "mkdir -p /tmp/c1-export"

kubectl exec -i -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "cat > /tmp/c1-export/${ARCHIVE}" < "${ARCHIVE}"

kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "ls -lh /tmp/c1-export/${ARCHIVE}"
```

---

## Alternative with `kubectl cp`

This may work, but depends on whether the Jira image has `tar` installed.

```bash
kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- mkdir -p /tmp/c1-export

kubectl cp "${OUT}.tar.gz" "${TARGET_NS}/${JIRA_POD}:/tmp/c1-export/${OUT}.tar.gz"
```

If `kubectl cp` fails, use the safer `cat > file` method above.

---

## To download it back later from Jira pod

```bash
kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "cat /tmp/c1-export/${OUT}.tar.gz" > "./${OUT}-from-jira-pod.tar.gz"
```

---

## Important warning

This archive contains sensitive data if you export:

```text
secrets/_SENSITIVE_all-secrets-full.yaml
```

It can include:

- database passwords
- Jira/Confluence secrets
- Docker registry credentials
- TLS private keys
- tokens

So after analysis, delete it from the pod:

```bash
kubectl exec -n "${TARGET_NS}" "${JIRA_POD}" -- sh -c "rm -rf /tmp/c1-export"
```
