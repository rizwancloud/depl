# Registration App CI/CD and GitOps — Project Documentation

## 1. Project overview

This project automates delivery of a Java/Maven registration application to a Kubernetes cluster. Jenkins runs Continuous Integration (CI), builds a versioned Docker image, and triggers a separate Continuous Delivery (CD) job. The CD job commits an image update to GitHub; Argo CD watches that repository and synchronizes Kubernetes to match the Git-defined desired state.

### Source repositories

| Purpose | Repository | Branch / file |
|---|---|---|
| Application source and CI pipeline | https://github.com/rizwancloud/registration_app_code | `master`, `Jenkinsfile` |
| Kubernetes manifests and CD pipeline | https://github.com/rizwancloud/k8s_registe_app | `main`, `JenkinsFile` |

## 2. Architecture and flow

```text
Developer change
      |
      v
GitHub: registration_app_code (master)
      |
      v
Jenkins: pipeline-for-CI (agent label: agent)
  - Checkout source
  - Maven build and tests
  - SonarQube analysis / Quality Gate result
  - Build Docker image
  - Push image to Docker Hub
  - Trivy image vulnerability scan
      |
      | IMAGE_NAME + IMAGE_TAG
      v
Jenkins: pipeline-for-CD
  - Checkout k8s_registe_app (main)
  - Update deployment.yaml image tag
  - Commit and push manifest change
      |
      v
GitHub: k8s_registe_app (main)
      |
      v
Argo CD: register-app (automatic sync, prune, self-heal)
      |
      v
Kind Kubernetes cluster
  Deployment myapp2 -> ReplicaSet -> 3 Pods
      |
      v
Service myapp2-service (port 80 -> container port 8080)
      |
      v
Registration app (context path shown as /webapp/)
```

## 3. Infrastructure used

The setup notes describe four EC2 instances:

| Instance | Responsibility |
|---|---|
| `jenkins-master` | Jenkins controller / web UI and pipeline orchestration. |
| `jenkins-agent` | Executes build stages and tools required by the CI/CD pipelines. |
| `sonarqube` | SonarQube code analysis, backed by PostgreSQL. |
| `k8s-argocd` | Docker, Kind multi-node cluster, kubectl, and Argo CD. |

The screenshots supplied with the project show the instances running and passing EC2 status checks. Instance sizes and public IPs are environment-specific; do not hard-code them in documentation or pipeline files.

## 4. Set up Jenkins controller and agent

### Jenkins controller

The supplied `jenkins-setup.sh` targets Amazon Linux 2023. It updates packages, installs Java 21 and Git, adds the Jenkins stable repository, installs Jenkins, enables and starts the service, and prints the initial administrator password.

After running it:

1. Open `http://<jenkins-public-ip>:8080` and complete the initial Jenkins setup.
2. In the EC2 security group, allow TCP `8080` only from trusted IP addresses. Avoid unrestricted public access.
3. Install the pipeline, Git, credentials, SonarQube, Docker-related, and other plugins required by the configured jobs.

### Jenkins agent

The setup notes say the worker was configured with Git, Maven, Java, and Docker. In Jenkins, add/connect the worker as a node with label `agent`, then run the launch command shown by Jenkins on the worker. The CI pipeline requests this label.

A `/tmp` remount with a 2 GB size was used during troubleshooting when Jenkins nodes appeared offline. Treat this as a runtime workaround; check `/tmp` capacity and agent logs if the issue returns, and make any required persistent OS change deliberately.

Confirm that the controller can run a simple test pipeline and that the agent is online before configuring the application pipeline.

## 5. Set up SonarQube

The supplied setup notes describe installing Java 21, PostgreSQL, and Git; creating a PostgreSQL role/database; configuring OS limits and `vm.max_map_count`; installing SonarQube under `/opt/sonarqube`; and running it through a systemd service.

Key configuration checks:

- Configure the JDBC username, password, and URL in `/opt/sonarqube/conf/sonar.properties` to match the PostgreSQL role and database actually created.
- **Database-name check:** the notes create a database named `sonarqube`, but a later example JDBC URL references `sonarqubedb`. Those names must match. If the database is still named `sonarqube`, the URL should use `jdbc:postgresql://localhost/sonarqube`.
- Ensure PostgreSQL authentication permits the configured local database user/password method, and inspect the SonarQube and PostgreSQL logs if startup fails.
- Start and enable the SonarQube systemd service, then open `http://<sonarqube-public-ip>:9000` only from trusted IPs.
- Create a SonarQube token and store it in Jenkins credentials. Configure the SonarQube server in Jenkins with the exact name `sonarqube` because the CI pipeline calls `withSonarQubeEnv('sonarqube')`.
- Configure a webhook in SonarQube to `http://<jenkins-public-ip>:8080/sonarqube-webhook/` so Jenkins can receive the Quality Gate result.

Do not put the PostgreSQL password, SonarQube token, or any other secret in Git. A password appeared in the supplied setup notes; if it was a real value, rotate it and use a new secret in PostgreSQL and SonarQube configuration.

## 6. Set up Kind and Argo CD

The setup notes describe installing Docker, Kind, and kubectl on the `k8s-argocd` EC2 instance, then creating a cluster named `dev-cluster` with one control-plane node and two worker nodes. Example `kind-multi-node.yaml`:

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: dev-cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Create the cluster using the file and verify that its nodes are ready. The supplied setup used `sudo` for Kind and kubectl; continue using the same user context (or deliberately configure your own kubeconfig) so kubectl points at the cluster you created:

```bash
sudo kind create cluster --name dev-cluster --config kind-multi-node.yaml
sudo kubectl get nodes
```

Install Argo CD into its namespace and wait for the pods to become ready:

```bash
sudo kubectl create namespace argocd
sudo kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sudo kubectl get pods -n argocd
sudo kubectl get svc -n argocd
```

For the lab setup, the notes expose the Argo CD server using a port-forward:

```bash
sudo kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

Open `https://<kind-ec2-public-ip>:8080`. Retrieve the initial admin password with:

```bash
sudo kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Keep the port-forward process running while accessing the UI. Binding to `0.0.0.0` exposes the forwarded port on the EC2 interface, so restrict the security-group rule to trusted IPs. For a production setup, use a deliberately configured ingress/load balancer and TLS rather than relying on a temporary port-forward.

### Argo CD application configuration

Create an application using these values:

| Setting | Value |
|---|---|
| Application name | `register-app` |
| Project | `default` |
| Repository | `https://github.com/rizwancloud/k8s_registe_app.git` |
| Path | `./` |
| Target revision | `HEAD` / `main` |
| Destination | `in-cluster` |
| Namespace | `default` |
| Sync policy | Automatic |
| Automated options | Enable sync, prune resources, and self-heal |

## 7. CI pipeline — `pipeline-for-CI`

The CI `Jenkinsfile` is stored in `registration_app_code` and uses the Jenkins node label `agent`.

1. **Cleanup Workspace:** removes previous workspace files.
2. **Checkout from SCM:** checks out the configured repository revision.
3. **Build Application:** runs `mvn clean package`.
4. **Test Application:** runs `mvn test`.
5. **SonarQube Analysis:** runs Maven verification and the Sonar Maven scanner.
6. **Quality Gate:** waits up to five minutes for SonarQube. The current file uses `abortPipeline: false`, so a failed Quality Gate result does not automatically stop the pipeline.
7. **Docker Build / Push:** builds and publishes the image to Docker Hub.
8. **Trivy Image Scan:** scans for HIGH and CRITICAL vulnerabilities using `--exit-code 1`; findings at those severities fail the stage.
9. **Cleanup Artifacts:** removes the built local image.
10. **Trigger CD Pipeline:** starts `pipeline-for-CD` and passes `IMAGE_NAME` and `IMAGE_TAG`.

### Image name

The current CI file sets `APP_NAME` to `register-app` and constructs `IMAGE_NAME` as `rizwanregister-app`. The image pushed by the Docker stages is:

`admindockerrizwan/rizwanregister-app:<BUILD_NUMBER>`

### Required Jenkins configuration

- Jenkins agent with Java, Maven, Git, Docker and Trivy, with permission to access the Docker daemon.
- SonarQube server configuration named `sonarqube` and a working webhook to `/sonarqube-webhook/`.
- Docker Hub username/password or access-token credential with ID `dockerhub-credentials`.
- Git credential with ID `github` for the CD push.
- A downstream job named exactly `pipeline-for-CD`, accepting string parameters `IMAGE_NAME` and `IMAGE_TAG`.

The current CI file includes `DOCKER_PASS = 'dockerhub'` as an environment value, but the Docker login stage uses Jenkins Credentials (`dockerhub-credentials`). Treat the environment value as a placeholder, remove it if unused, and never put a real Docker Hub password/token in the Jenkinsfile.

## 8. CD pipeline and Kubernetes resources

The CD pipeline is stored in `k8s_registe_app/JenkinsFile`. It checks out the repository, updates the image reference in `deployment.yaml`, commits the change, and pushes to `main`. Argo CD then reconciles the cluster from the Git repository.

| Resource | Current configuration |
|---|---|
| Deployment | `myapp2`, 3 replicas, label/selector `app: myapp2`, container port `8080`. |
| Service | `myapp2-service`, type `NodePort`, port `80`, target port `8080`, NodePort `30080`. |
| Argo CD app | `register-app`, repository `k8s_registe_app`, path `./`, namespace `default`, automatic sync. |

### Image parameter consistency

The current CI job passes `IMAGE_NAME=rizwanregister-app`, and the checked-in manifest uses an image under `admindockerrizwan/rizwanregister-app`. The CD script's `sed` expression matches the `rizwanregister-app` portion when CI supplies that value. However, the CD job's manual default is `admindockerrizwan/register-app`, which does **not** match the current manifest image. Prefer running CD through CI so it receives the intended parameters, or align the default and replacement logic before running CD manually.

### Verify deployment health

Run from a machine whose kubectl context points to the Kind cluster:

```bash
sudo kubectl get nodes
sudo kubectl get deployment myapp2
sudo kubectl get pods -l app=myapp2
sudo kubectl get service myapp2-service
sudo kubectl rollout status deployment/myapp2
sudo kubectl logs -l app=myapp2 --tail=100
```

The screenshots supplied with the project show the Argo CD application as **Healthy** and **Synced**, with three application Pods running. They also show a successful SonarQube Quality Gate and successful recent Jenkins CI/CD builds.

## 9. Accessing the application

The Service manifest exposes NodePort `30080`. The provided lab access method forwards the Service port to host port `80`:

```bash
sudo kubectl port-forward svc/myapp2-service 80:80 --address 0.0.0.0
```

Then open `http://<kind-ec2-public-ip>/webapp/` while the port-forward is running and the EC2 security group permits the relevant inbound traffic. Direct access through the NodePort is a separate method and uses `http://<reachable-node-ip>:30080/webapp/` if routing and firewall rules permit it.

Port-forwarding is temporary and ends when its process stops. Do not treat it as a production ingress solution. For public production access, configure an appropriate load balancer or ingress, TLS, and narrowly scoped network rules.

## 10. Credentials and security checklist

- Keep Docker Hub credentials, GitHub tokens, SonarQube tokens, database passwords, and Argo CD admin credentials out of source control.
- Use the Jenkins Credentials store and reference credentials by ID from pipeline code.
- Restrict inbound ports `8080` (Jenkins / Argo CD lab forward), `9000` (SonarQube), `80`, and `30080` to trusted source addresses as needed. Avoid unrestricted administrative interfaces.
- Rotate any secret that was shared in notes, screenshots, or committed files.
- Prefer TLS, least-privilege credentials, pinned container versions, and managed ingress for a real deployment.

## 11. Troubleshooting

| Symptom | What to check |
|---|---|
| Jenkins agent offline | Check the node's launch command, Java/agent logs, connectivity to the controller, disk space and `/tmp` capacity. |
| Maven build fails | Check Java/Maven versions, dependencies, repository access and the Jenkins agent workspace. |
| Docker build/push fails | Check Docker daemon permissions, login credential ID `dockerhub-credentials`, image namespace and registry connectivity. |
| SonarQube does not start | Check Java, PostgreSQL status, JDBC username/password/URL, OS limits, `vm.max_map_count`, and SonarQube logs. |
| Quality Gate times out | Check the Jenkins SonarQube server name, token configuration, and SonarQube webhook URL/reachability. |
| Trivy stage fails | Review the HIGH/CRITICAL findings. The current `--exit-code 1` setting intentionally returns failure when such findings exist. |
| CD commits no change | Check `IMAGE_NAME` and `IMAGE_TAG`; verify the current manifest image matches the CD script's string-replacement pattern. |
| Argo CD is OutOfSync | Check repository URL, revision, path, manifest validity, and application sync events. |
| Pods are not ready | Run `kubectl describe pod`, inspect pod events, check image pull status and view logs. |
| App cannot be opened | Check Pod readiness, Service selector/ports, active port-forward, application context path `/webapp/`, and EC2 security-group rules. |

## 12. Reference setup script

The file `jenkins-setup.sh` supplied with the project installs Jenkins on Amazon Linux 2023 using Java 21 and prints the initial administrator password. Store that script with your setup notes if you want to reproduce the controller installation. Restrict the Jenkins port to trusted IP addresses before using the UI.
