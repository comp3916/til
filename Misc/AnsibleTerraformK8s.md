Below is an example of how you can use **Terraform Cloud** to provision the infrastructure (Azure Kubernetes Service - AKS) and **Ansible** to configure the AKS cluster and deploy a Spring Boot microservices-based application from a Git repository.

---

### **Step 1: Use Terraform Cloud to Provision AKS**
Terraform Cloud will be used to create the AKS cluster and other necessary Azure resources (e.g., resource group, networking).

#### **Terraform Configuration**
Create a `main.tf` file for provisioning the AKS cluster:

```hcl
# main.tf
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "myResourceGroup"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "myAKSCluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "myakscluster"

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }

  service_principal {
    client_id     = var.client_id
    client_secret = var.client_secret
  }
}

output "kubeconfig" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
```

#### **Terraform Cloud Setup**
1. Create a workspace in Terraform Cloud.
2. Upload the `main.tf` file to the workspace.
3. Set the following variables in Terraform Cloud:
   - `client_id`: Azure service principal client ID.
   - `client_secret`: Azure service principal client secret.
4. Trigger a Terraform run to provision the AKS cluster.

---

### **Step 2: Use Ansible to Configure AKS and Deploy the Spring Boot Application**
Once the AKS cluster is provisioned, use Ansible to:
1. Configure `kubectl` to connect to the AKS cluster.
2. Deploy the Spring Boot microservices application from a Git repository.

#### **Ansible Playbook**
Create a playbook (`deploy_springboot.yml`) to manage the AKS cluster and deploy the application:

```yaml
---
- name: Deploy Spring Boot Microservices to AKS
  hosts: localhost
  connection: local
  vars:
    git_repo: "https://github.com/your-repo/springboot-microservices.git"
    kubeconfig: "{{ lookup('env', 'KUBECONFIG') }}"
  tasks:
    - name: Install kubectl
      become: yes
      apt:
        name: kubectl
        state: present

    - name: Write kubeconfig to file
      copy:
        content: "{{ kubeconfig }}"
        dest: ~/.kube/config
        mode: '0600'

    - name: Verify cluster connection
      command: kubectl get nodes
      register: result
      ignore_errors: yes

    - name: Fail if cluster connection fails
      fail:
        msg: "Failed to connect to AKS cluster"
      when: result.rc != 0

    - name: Clone Spring Boot microservices repository
      git:
        repo: "{{ git_repo }}"
        dest: ./springboot-microservices
        version: main  # Replace with your branch/tag

    - name: Deploy Spring Boot application using kubectl
      kubernetes.core.k8s:
        state: present
        src: ./springboot-microservices/k8s/deployment.yaml  # Path to your Kubernetes manifests
      environment:
        KUBECONFIG: ~/.kube/config
```

---

### **Step 3: Kubernetes Manifests for Spring Boot Application**
Ensure your Git repository contains Kubernetes manifests for deploying the Spring Boot application. For example:

#### **deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot-app
  template:
    metadata:
      labels:
        app: springboot-app
    spec:
      containers:
        - name: springboot-app
          image: your-docker-repo/springboot-app:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: springboot-app
```

---

### **Step 4: Run the Ansible Playbook**
1. Export the kubeconfig from Terraform Cloud output:
   ```bash
   export KUBECONFIG=$(terraform output -raw kubeconfig)
   ```
2. Run the Ansible playbook:
   ```bash
   ansible-playbook deploy_springboot.yml
   ```

---

### **What Happens in the Playbook**
1. **Install kubectl**: Ensures `kubectl` is installed on the local machine.
2. **Write kubeconfig**: Saves the kubeconfig file to `~/.kube/config`.
3. **Verify Cluster Connection**: Checks if the AKS cluster is accessible.
4. **Clone Git Repository**: Clones the Spring Boot microservices repository.
5. **Deploy Application**: Applies the Kubernetes manifests to deploy the Spring Boot application.

---

### **Step 5: Access the Spring Boot Application**
Once the playbook completes, the Spring Boot application will be deployed to the AKS cluster. You can access it via the external IP of the `springboot-service`:

```bash
kubectl get svc springboot-service
```

---

### **Summary**
- **Terraform Cloud**: Provisions the AKS cluster and other Azure infrastructure.
- **Ansible**: Configures the AKS cluster and deploys the Spring Boot microservices application from a Git repository.

This approach combines the strengths of both tools:
- Terraform for **infrastructure as code**.
- Ansible for **configuration management** and **application deployment**.

