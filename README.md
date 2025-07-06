
# 🌀 ArgoCD Stuck in “Refreshing…” State — Troubleshooting & Resolution

## 📌 Problem

 ArgoCD applications (e.g., `forecast-dev`) appeared to be stuck in the **"Refreshing…"** state in both the **Web UI** and **CLI**:

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

Restart the repo server as well:

```bash
kubectl -n argocd rollout restart deployment argocd-repo-server
```

Wait 30–60 seconds for pods to restart.

---

### ✅ Step 4: Retry Refresh

```bash
argocd app get forecast-dev --refresh
```

Or refresh from the UI.

> ✅ **Expected result:** The app should now respond normally and leave the "Refreshing..." state.

---

## 📷 Optional Screenshots

<details>
  <summary>🔍 Stuck UI Before Fix</summary>

![Stuck Refreshing UI](./images/stuck-refreshing-ui.png)

</details>

<details>
  <summary>✅ App After Restart</summary>

![App Unstuck](./images/app-after-restart.png)

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
