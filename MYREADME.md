# Setup
Please watch the GitHub CLI (gh) - How to manage repositories more efficiently video if you are not familiar with GitHub CLI.

gh auth login --web

gh repo fork vfarcic/argo-events-gh-demo --clone --remote

cd argo-events-gh-demo

gh repo set-default


# k8s & argo setup
colima start --cpu 4 --memory 8 --mount $HOME:w || colima kubernetes start 

kubectl create namespace a-team

helm upgrade --install argocd argo-cd \
    --repo https://argoproj.github.io/argo-helm \
    --namespace argocd --create-namespace \
    --values values-argocd.yaml --wait

kubectl port-forward service/argocd-server -n argocd 8080:443 >/dev/null 2>&1 &

REPO_URL=$(git config --get remote.origin.url)

yq --inplace ".spec.source.repoURL = \"$REPO_URL\"" \
    apps-argocd.yaml

yq --inplace ".spec.source.repoURL = \"$REPO_URL\"" \
    apps/silly-demo.yaml

git add .

git commit -m "Make it my own"

git push

kubectl apply --filename apps-argocd.yaml

helm upgrade --install argo-events argo-events \
    --repo https://argoproj.github.io/argo-helm \
    --namespace argo-events --create-namespace \
    --set webhook.enabled=true --wait

kubectl --namespace argo-events apply \
    --filename https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml

export GITHUB_TOKEN=[...]
curl -sS -f -I -H "Authorization: token $GITHUB_TOKEN" https://api.github.com | grep -i x-oauth-scopes

kubectl --namespace argo-events \
    create secret generic github \
    --from-literal token="token $GITHUB_TOKEN"

kubectl get secret -nargo-events github -o jsonpath="{.data.token}" | base64 -d    

export GITHUB_USER=[...]

yq --inplace \
    ".spec.triggers[0].template.http.url = \"https://api.github.com/repos/$GITHUB_USER/argo-events-gh-demo/dispatches\"" \
    sensor-deployment.yaml

gh repo view --web    


# Argo Events Setup
kubectl apply --filename sa.yaml

kubectl --namespace argo-events apply \
    --filename event-source-deployment.yaml

kubectl --namespace argo-events apply \
    --filename sensor-deployment.yaml

# Combining Workflow Runs and GitOps
Modify main.go 

git add .

git commit -m "Silly demo"

git push

gh repo view --web