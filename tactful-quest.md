# Setup & Deployment Instructions
## Docker-Compose
First you have to run `docker-compose up --build` command to spin up all the services and DB and redis
we have two networks : frontend (votes and results exposed) and backend worker,redis and db not exposed
after visiting localhost:8080 and localhost:8081 to vote and check the results we can spin up the seed profile to see how our voting app deals with high volume of traffic
we will scale workers to 10 using this command 
`docker compose  --profile seed up --scale worker=10  -d`

## MiniKube
- used minikube as it is my main toold for testing local k8s cluster apps.
- used sealed-secrets to safely store secrets encrypted and be able to push them to github
- used keda (k8s event driven auto-scaling) to scale workers when redis has over 30 items in the votes list
- used kustomize to reuse manifests files from base folder and override or add resources to the dev/prod environment.
I can use kubectl apply -k ./k8s/prod on the other minikube cluster and I would have all the resources I had in my original cluster alongside the edits in the `prod` folder
- used calico cni to make network policy take effect and deeny all other traffic except from result and worker
- used nginx-ingres controller to forward traffic from domain name to corresponding services
- Enforced non-root policies (PSA) for all services and used appropriate resource limits and liveness and readiness probes


```bash
minikube start -p dev --kubernetes-version='stable' --nodes=1 --driver='docker' --disk-size='20000mb'  --cpus='2' --base-image='gcr.io/k8s-minikube/kicbase:v0.0.48@sha256:7171c97a51623558720f8e5878e4f4637da093e2f2ed589997bedc6c1549b2b1' --memory='4g' --cni=calico

minikube -p dev addons enable ingress
minikube -p dev tunnel
```

```text
# /etc/hosts file 
127.0.0.1 vote-dev.tactful-ai-voting-app.com results-dev.tactful-ai-voting-app.com vote-prod.tactful-ai-voting-app.com results-prod.tactful-ai-voting-app.com
```
```bash
minikube -p dev image load tactful-ai-voting-app-seed
minikube -p dev image load tactful-ai-voting-app-vote
minikube -p dev image load tactful-ai-voting-app-result
minikube -p dev image load tactful-ai-voting-app-worker

kubectl apply -k ./k8s/dev
```

### testing networking policies
```bash
# this command will work
kubectl run test --rm -it --image=alpine --namespace=dev --labels="app=worker" -- sh
# netcat to test whether TCP port is open on DB pod without sending data
nc -vz 10.244.90.136 5432
# this command will not connect to DB pod because of networkpolicy
kubectl run test --rm -it --image=alpine --namespace=dev --labels="app=test" -- sh
nc -vz 10.244.90.136 5432
```

here are the commands for creating sealed secrets, this is good for local development, but if I were to use Azure or any other cloud I would go with External-Secrets Operator (ESO)

```bash
kubectl create secret generic postgres-dot-net-secret --from-literal=DATABASE_URL="Server=db;Username=postgres;Password=postgres;Database=postgres;" --namespace dev  --dry-run=client -o yaml > secret.yaml

# for linux terminal
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
# for powershell terminal
Get-Content secret.yaml | kubeseal --format yaml --controller-namespace dev | Set-Content ./k8s/dev/postgres-dot-net-secret.yaml 

kubectl create secret generic postgres-secret --from-literal=DATABASE_URL="postgres://postgres:postgres@db/postgres" --namespace dev  --dry-run=client -o yaml > secret.yaml
Get-Content secret.yaml | kubeseal --format yaml --controller-namespace dev | Set-Content ./k8s/dev/postgres-secret.yaml 

### PROD
kubectl create secret generic postgres-secret --from-literal=DATABASE_URL="postgres://postgres:postgres@db/postgres" --namespace prod  --dry-run=client -o yaml > secret.yaml
Get-Content secret.yaml | kubeseal --format yaml --controller-namespace prod | Set-Content ./k8s/prod/postgres-secret.yaml 
 

kubectl create secret generic postgres-dot-net-secret --from-literal=DATABASE_URL="Server=db;Username=postgres;Password=postgres;Database=postgres;" --namespace prod  --dry-run=client -o yaml > secret.yaml
Get-Content secret.yaml | kubeseal --format yaml --controller-namespace prod | Set-Content ./k8s/prod/postgres-dot-net-secret.yaml 
```