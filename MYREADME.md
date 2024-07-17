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
open http://localhost:8080

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


# We can confirm that’s what really happened by pulling the latest changes from Git.

git pull
We should see from the output that silly-demo/deployment.yaml was modified, thus confirming that the first workflow run was successful.

# From now on, Argo CD should detect the change and we should see new Pods deployed to the cluster.

kubectl --namespace a-team get pods
:(

## troubleshooting...
stern argocd-repo-server -nargocd

argocd-repo-server-5d8cc48795-z8tzs repo-server time="2024-07-16T18:16:27Z" level=error msg="finished unary call with code Unknown" error="error creating SSH agent: \"SSH agent requested but SSH_AUTH_SOCK not-specified\"" grpc.code=Unknown grpc.method=GenerateManifest grpc.service=repository.RepoServerService grpc.start_time="2024-07-16T18:16:27Z" grpc.time_ms=2.848 span.kind=server system=grpc

kubectl config set-context --current --namespace=argocd
argocd repo add https://github.com/mitin20/argo-events-gh-demo --username mitin20 --password $GITHUB_TOKEN
argocd app list
argocd app get apps

ComparisonError  Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"  2024-07-16 13:00:41 -0500 -05

argocd app sync apps

FATA[0000] rpc error: code = FailedPrecondition desc = error resolving repo revision: rpc error: code = Unknown desc = error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"

argocd admin export
    message: 'Failed to load target state: failed to generate manifest for source
      1 of 1: rpc error: code = Unknown desc = error creating SSH agent: "SSH agent
      requested but SSH_AUTH_SOCK not-specified"'

eval `ssh-agent -s`
ssh-add ~/.ssh/id_rsa_m1
argocd repo add git@github.com:mitin20/argo-events-gh-demo.git --ssh-private-key-path ~/.ssh/id_rsa_m1

error testing repository connectivity: ssh: this private key is passphrase protected

ssh-keygen -t ed25519 -C "mitin20@gmail.com"
eval "$(ssh-agent -s)"
vim ~/.ssh/config

Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519

ssh-add --apple-use-keychain ~/.ssh/id_ed25519
pbcopy < ~/.ssh/id_ed25519.pub

argocd repo add git@github.com:mitin20/argo-events-gh-demo.git --ssh-private-key-path ~/.ssh/id_ed25519

argocd app sync apps

kubectl --namespace a-team get pods
:)


to be continued...



# To exit the Devbox shell and return to your regular shell:

exit
