# Auto-Rollback Attempts

> Content: Attempts to enable auto-rollback to the last "Healthy" commit when the app status is "Degraded".

## Approach 1:
Create an ArgoCD pod and run the login command in it to connect to your ArgoCD app. Then check the app status continuously from inside the pod. If the status is not "Healthy" or "Progressing", perform rollback.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: argocd-health-check
  labels:
    app: argocd-health-check
spec:
  containers:
  - name: argocd-cli
    image: argoproj/argocd:v2.6.15  # Replace with the required ArgoCD CLI version
    command: ["/bin/sh", "-c"]
    args:
      - |
        # Log in to ArgoCD
        APP_NAME= <app-name>
        argocd login <IP:port> --username <username> --password <password> --insecure;
        # Define the application name
        sleep 30
        STATUS=$(argocd app get $APP_NAME  | awk '/Health Status/ {print $3}')
        echo "Application health status: $STATUS"
        while true; do
          echo "Application health status: $STATUS"
          if [ "$STATUS" != "Healthy" ] && [ "$STATUS" != "Progressing" ]; then
            argocd app set $APP_NAME --sync-policy none
            echo "Application is not healthy. Performing rollback..."
            # Get the revision history
            revisions=$(argocd app history $APP_NAME | awk 'NR>1 {print $NF}' | sed 's/[()"]//g')
            # Iterate through revisions and check health
            for revision_id in $revisions; do
            echo "Checking revision ID: $revision_id"

            # Sync to the revision in dry-run mode
            output=$(argocd app sync $APP_NAME --revision $revision_id --dry-run --prune --force | awk '/Health Status/ {print $3}')

            echo "Output: $output"
            # Check if the health status is Healthy
            if [ "$output" = "Healthy" ]; then
              echo "Found healthy revision: $revision_id"                
    
              # Rollback to this revision
              argocd app rollback $APP_NAME "$revision_id"
              echo "Rolled back to healthy revision: $revision_id"
              exit 0
            fi
          done
          else
            echo "Application is healthy. No rollback needed."
          fi

          # Sleep for 5 seconds to avoid constant looping
          sleep 5
        done
```
<br>

```bash
argocd login <IP:port> --username <username> --password <password> --insecure;
```
> The command above is used to connect to ArgoCD.
<br>


```bash
STATUS=$(argocd app get $APP_NAME  | awk '/Health Status/ {print $3}')
```
> The command above assigns Health Status value of the app to STATUS variable.


 ```bash
 argocd app set $APP_NAME --sync-policy none
 ```
 > To perform rollback we have to disable auto-sync if it is enabled by running the command above.

```bash
revisions=$(argocd app history $APP_NAME | awk 'NR>1 {print $NF}' | sed 's/[()"]//g')
```
> revisions is the array of revision hashes of the old commits.

```bash
for revision_id in $revisions; do
    echo "Checking revision ID: $revision_id"

    # Sync to the revision in dry-run mode
    output=$(argocd app sync $APP_NAME --revision $revision_id --dry-run --prune --force | awk '/Health Status/ {print $3}')

    echo "Output: $output"
    # Check if the health status is Healthy
    if [ "$output" = "Healthy" ]; then
        echo "Found healthy revision: $revision_id"                
    
        # Rollback to this revision
        argocd app rollback $APP_NAME "$revision_id"
        echo "Rolled back to healthy revision: $revision_id"
        exit 0
    fi
done
```
> Above, tries the revisions in a loop and checks the status of these revisions without applying the changes to app by using --dry-run option. If the revision status is "Healthy" performs rollback to this revision.

## Issues of Approach 1: 
Example output of "sync --dry-run":
```txt
Name:               argocd/<app>
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          <app>
URL:                <url>
Repo:               <repo>
Target:             HEAD
Path:               manifests
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to HEAD (f0c3382)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME                 STATUS  HEALTH   HOOK  MESSAGE
       Service     <app>       client               Synced  Healthy        service/client configured (dry run)
       Service     <app>       nginx1               Synced  Healthy        service/nginx1 configured (dry run)
       Service     <app>       server               Synced  Healthy        service/server configured (dry run)
       Service     <app>       nginx2               Synced  Healthy        service/nginx2 configured (dry run)
       Pod         <app>       nginx2               Synced  Healthy        pod/nginx2 configured (dry run)
       Pod         <app>       nginx1               Synced  Healthy        pod/nginx1 configured (dry run)
       Pod         <app>       argocd-health-check  Synced  Healthy        pod/argocd-health-check configured (dry run)
apps   Deployment  <app>       server-deployment    Synced  Healthy        deployment.apps/server-deployment configured (dry run)
apps   Deployment  <app>       client-deployment    Synced  Healthy        deployment.apps/client-deployment configured (dry run)

```
- The output of --dry-run option shows the Health Status of the current commit, so we can't check the resulting status of the revision.
- The --dry-run option only tests wether the pods can be created properly it doesn't check if the containers properly works after that. 

## Approach 2 

> There will be an ArgoCD pod and a git pod that tags each commit as "Healthy" if the app status is healthy or "Degraded" if the app status is degraded. Then when a rollback needed, the git pod searches for an older commit that has tagged "Healthy" and ArgoCD will sync with that commit.

## Issues of Approach 2
- ArgoCD will send the status of the app to git pod and git pods will send the "Healthy" commit hash to ArgoCD pod so, ArgoCD and git pods have to communicate and this is an extra work.
- It modifies the main repository.
