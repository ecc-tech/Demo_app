# Ansible Deployment Playbook: Sequence and Workflow

Based on the structure and files in your `ansible` directory, here is the sequence for running the playbook, followed by an explanation of the workflow and code.

## 1. Sequence of Running the Playbook

To trigger the playbook (typically from Jenkins or a CI/CD pipeline), you need to execute the main deploy playbook ([ansible/playbooks/deploy.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/playbooks/deploy.yml)) and provide the necessary inventory and extra variables (like the Docker image tag).

**Command to run:**
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/deploy.yml -e "image_tag=<YOUR_IMAGE_TAG>"
```
*Note: If `image_tag` is not passed via the `-e` flag, the playbook will fail during its `pre_tasks` validation unless run interactively (where it prompts for it).*

## 2. Code and Workflow Explanation

The deployment workflow is orchestrated by the [deploy.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/playbooks/deploy.yml) playbook, which primarily invokes the `eks-deploy` role. The `eks-deploy` role contains a modular set of tasks to ensure a safe, automated, and resilient deployment to AWS EKS.

### Overall Workflow (The `eks-deploy` Role)
The master sequence is defined in [roles/eks-deploy/tasks/main.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/tasks/main.yml) and consists of 5 clear steps:

#### Step 1: Ensure Namespace Exists
- Uses the `kubernetes.core.k8s` module to ensure the target Kubernetes namespace (e.g., `demoapp-dev`) exists.
- It applies labels to the namespace indicating it is managed by Ansible.

#### Step 2: Configure Kubeconfig for EKS
- Runs the AWS CLI command `aws eks update-kubeconfig` to fetch and update the local `~/.kube/config` file.
- This ensures that Ansible and `kubectl` can authenticate and communicate with the correct EKS cluster (`Data-Oceano-devCluster`).

#### Step 3: Deploy Application ([deploy.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/playbooks/deploy.yml))
Applies the Jinja2 ([.j2](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/templates/pvc.yaml.j2)) templates defined in the `templates/` directory to create/update Kubernetes resources:
1. **PVC ([pvc.yaml.j2](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/templates/pvc.yaml.j2))**: Provisions a PersistentVolumeClaim for storage.
2. **Deployment ([deployment.yaml.j2](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/templates/deployment.yaml.j2))**: Deploys the application pods using the `image_tag` passed during execution. It is configured for zero-downtime rolling updates.
3. **Service ([service.yaml.j2](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/templates/service.yaml.j2))**: Provisions a ClusterIP service to expose the pods internally.
4. **Ingress ([ingress.yaml.j2](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/templates/ingress.yaml.j2))**: Conditionally creates an Ingress resource (AWS ALB) to route external traffic to the service, depending on the `ingress_enabled` variable.

#### Step 4: Verify Rollout ([verify.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/tasks/verify.yml))
- Executes `kubectl rollout status` to actively monitor the progress of the new deployment.
- It waits for a specified timeout (default `300s`).
- Instead of failing the playbook immediately if the rollout stalls or fails, it gracefully captures the exit code and sets a dynamic variable flag `rollout_failed: true`.

#### Step 5: Automatic Rollback ([rollback.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/tasks/rollback.yml))
- This step acts as a safety net and **only runs if `rollout_failed` is true**.
- If the new deployment fails health checks or times out, it executes `kubectl rollout undo` to revert the application back to the previous stable state.
- It waits for the rollback itself to finish and then purposefully fails the Ansible playbook (`ansible.builtin.fail`). This ensures the CI/CD pipeline (e.g., Jenkins) accurately reports a failed build.

### Variables Configuration
The entire workflow is highly parameterized. Default values are stored in [roles/eks-deploy/vars/main.yml](file:///home/crio-sys/ECC-work/Data_Oceano_system/ansible/roles/eks-deploy/vars/main.yml), which handles:
- **AWS contexts**: Region, Account ID, ECR Registry.
- **App details**: App name, namespace, container ports, replicas.
- **Deployment strategies**: Maximum surge/unavailable properties for rolling updates.
- **Health Probes**: Liveness and readiness endpoints, timeouts, and thresholds.
- **Resource Limits**: CPU and memory limits.
