
# 🌀 ArgoCD Stuck in “Refreshing…” State — Troubleshooting & Resolution
![Argo-1-e1630327305635-1](https://github.com/user-attachments/assets/7c474710-f6b8-4ece-bd10-e9efdbb74592)
![Argo](https://github.com/user-attachments/assets/7c474710-f6b8-4ece-bd10-e9efdbb74592){: width="300" height="auto"}
## 📌 Problem

 ArgoCD applications appeared to be stuck in the **"Refreshing…"** state in both the **Web UI** and **CLI**:

```bash
argocd app get forecast-dev --refresh
# → Hangs indefinitely
```

Even after successful syncs, the application showed no errors in UI but failed to exit the refresh state.
![Screenshot 2025-07-01 155259](https://github.com/user-attachments/assets/ae349ab3-8e0c-4b50-8965-01243446d741)

---

## 🧠 Root Cause

After investigation:

- All ArgoCD pods were **healthy** and running.
- The application's last operation had **succeeded**.
- The CLI was also **stuck**, confirming it wasn't a UI bug.
- The refresh operation was blocked internally, likely due to a stale internal state or controller/repo-server deadlock.

![Screenshot 2025-07-01 155355](https://github.com/user-attachments/assets/99745cb9-3157-494b-8746-15f25c660cf8) 

---

## 🛠️ Resolution Steps

### ✅ Step 1: Confirm All ArgoCD Pods Are Running

```bash
kubectl -n argocd get pods
```

Ensure all pods like `argocd-application-controller`, `argocd-repo-server`, and `argocd-server` are in `Running` state.

![pods of argo](https://github.com/user-attachments/assets/266ed689-dbe8-4c25-9eb4-42ff5dc565b0)

---

### ✅ Step 2: Check Application Status

```bash
kubectl -n argocd get application forecast-dev -o yaml
```

Verify that:

```yaml
status:
  operationState:
    phase: Succeeded
    message: successfully synced (all tasks run)
```

If there is **no active operation**, the refresh should not hang.

---

### ✅ Step 3: Restart Controller and Repo Server

Since ArgoCD controller is a StatefulSet:

```bash
kubectl -n argocd rollout restart statefulset argocd-application-controller
```
![statefulset argo](https://github.com/user-attachments/assets/1c897c80-3f97-4c59-aa02-86da1b300927)


Restart the repo server as well:

```bash
kubectl -n argocd rollout restart deployment argocd-repo-server
```
![commands restart](https://github.com/user-attachments/assets/1995b430-74b4-46f3-8e9e-748a4e89b0ca)

Wait 30 seconds for pods to restart.

---

### ✅ Step 4: Retry Refresh

```bash
argocd app get forecast-dev --refresh
```

Or refresh from the UI.

> ✅ **Expected result:** The app should now respond normally and leave the "Refreshing..." state.

---

## 📷 Screenshots

<details>
  <summary>🔍 Stuck UI Before Fix</summary>

![Screenshot 2025-07-01 155259](https://github.com/user-attachments/assets/4869d319-fe0e-4958-ba9d-0983deb7f4c4)

</details>

<details>
  <summary>✅ App After Restart</summary>

![image](https://github.com/user-attachments/assets/a2eebd66-4a07-493e-831e-9cffef9e0c98)

</details>

---

## 🔁 Optional: Debug Logs (If Issue Persists)

Check the controller and repo-server logs:

```bash
kubectl -n argocd logs statefulset/argocd-application-controller --tail=100
kubectl -n argocd logs deploy/argocd-repo-server --tail=100
```

Look for:
- `context deadline exceeded`
- `rpc error`
- `failed to refresh application`

---

## 📎 Notes

- This issue was observed in multiple clusters.
- Most likely caused by a race condition or internal queue block inside ArgoCD’s controller.
- Restarting affected components resolved the issue without downtime.

---

## 🧼 Preventative Suggestions

- Monitor ArgoCD with alerts on stuck operations.
- Consider periodic controller restarts if this becomes common.
- Ensure ArgoCD is running a **stable release** (e.g., `v2.9.x`).
