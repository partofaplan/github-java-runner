# GitHub Actions Runner on Kubernetes

Deploy a self-hosted GitHub Actions runner into a local Kubernetes cluster using the official `actions-runner` container image and a lightweight bootstrap script that handles registration, deregistration, and token lifecycle.

## What Gets Deployed
- **Namespace** `github-actions-runner` isolates runner resources.
- **ServiceAccount** `github-runner` to follow least-privilege principles (no cluster-wide RBAC granted by default).
- **ConfigMap** `github-runner-scripts` supplying a startup script that:
  - Requests short-lived registration & removal tokens through the GitHub REST API.
  - Registers the runner with configurable labels.
  - Deregisters cleanly when the pod terminates.
- **Deployment** `github-runner` running a single replica of `partofaplan/github-actions-runner-java:latest` (extends the official image with Temurin JDK 17 and Apache Maven).

All manifests live under `manifests/` and are bundled via Kustomize.

## Prerequisites
- A Kubernetes cluster reachable via `kubectl` (kind, k3d, minikube, etc.).
- `kubectl` v1.21+ with Kustomize (built-in since v1.14).
- A GitHub personal access token (PAT) with the required scope:
  - For a repository-scoped runner: `repo`.
  - For an organization-wide runner: `admin:org`.
- Optional for GitHub Enterprise Server:
  - `GITHUB_SERVER_URL` (e.g. `https://github.example.com`).
  - `GITHUB_API_URL` (e.g. `https://github.example.com/api/v3`).

## 1. Create the PAT Secret
> ⚠️ Never commit the secret manifest. Use `kubectl create secret` to generate it in-cluster.

```bash
kubectl create namespace github-actions-runner

kubectl -n github-actions-runner create secret generic github-runner-secret \
  --from-literal=GITHUB_PAT='<your-personal-access-token>'
```

If you prefer YAML, see `manifests/secret-pat.yaml.sample`, replace the placeholder, and `kubectl apply` it (remember to keep it out of version control).

## 2. Build & Push the Java Runner Image

A lightweight Dockerfile (`docker/Dockerfile`) extends the official runner image with OpenJDK 17 and Maven 3.9.6 pre-installed so Maven builds work out-of-the-box.

```bash
IMAGE_TAG=partofaplan/github-actions-runner-java:latest

docker build -t "${IMAGE_TAG}" docker/
docker push "${IMAGE_TAG}"
```

If you prefer a different tag (e.g., to version images), update both the push command and the image reference in `manifests/deployment.yaml`.

## 3. Configure the Deployment
Edit `manifests/deployment.yaml` to match your target:

- `GITHUB_OWNER`: GitHub user or organization login that owns the runner scope.
- `GITHUB_REPOSITORY`: Repository name (without owner). Leave blank for an org-wide runner.
- `RUNNER_LABELS`: Optional comma-separated custom labels.
- Optional:
  - `GITHUB_SERVER_URL`: Override the default `https://github.com` for GHES.
  - `GITHUB_API_URL`: Override the default `https://api.github.com`.

Example adjustments for a repository runner:

```yaml
        - name: GITHUB_OWNER
          value: "your-org"
        - name: GITHUB_REPOSITORY
          value: "your-repo"
```

## 4. Deploy the Runner

```bash
kubectl apply -k manifests/
```

Watch the pod come up and tail its logs:

```bash
kubectl -n github-actions-runner get pods
kubectl -n github-actions-runner logs deploy/github-runner -f
```

Within a few seconds the runner should appear in the GitHub UI under **Settings → Actions → Runners** for the configured scope.

## 5. Verify & Use
- Trigger a workflow that targets the self-hosted runner (e.g. `runs-on: [self-hosted, kubernetes]`).
- During job execution the runner pod will show log output; the job workspace lives in an ephemeral `emptyDir` volume mounted at `/actions-runner/_work`.

## Cleanup

```bash
kubectl delete -k manifests/
kubectl delete namespace github-actions-runner
```

The shutdown hook requests a removal token and deregisters the runner automatically; manual cleanup in the GitHub UI should not be required.

## Notes & Customization
- **Scaling**: Increase `spec.replicas` in `manifests/deployment.yaml` to run multiple identical runners. Each pod registers with its own name (pod name) and shares the same PAT.
- **Runner Size**: Adjust resource requests/limits to match workload expectations.
- **Debugging**: Set `RUNNER_DEBUG=true` (env var) to enable verbose logging from the bootstrap script.
- **Security**: Use the narrowest PAT scopes possible and rotate regularly. For production, consider storing credentials in an external secret manager.
- **Alternatives**: For larger fleets or dynamic scaling, explore the upstream [actions-runner-controller](https://github.com/actions-runner-controller/actions-runner-controller).
