
# OpenShift Automation using Ansible

This article explains how to use Ansible to create basic resources on OpenShift.

## Installing and Configuring Ansible

Use this [link](https://github.com/mnakib/Labs-Ansible/blob/main/Installing-Ansible-on-RHEL10.md) to go through the process of installing and configuring Ansible on RHEL10]


## Install a Single Node OpenShift (SNO) Cluster

## Integrate OpenShift to Ansible

To make Ansible aware of your OpenShift cluster and successfully create the resources defined in your playbook, you need to configure Ansible with the correct connection and authentication details.

### 1\. Install the `redhat.openshift` collection

First, ensure you have the `redhat.openshift` collection installed, as the `k8s` module is part of it.

```bash
ansible-galaxy collection install redhat.openshift
```

### 2\. Configure Ansible to connect to OpenShift

The `redhat.openshift.k8s` module (and `community.okd.k8s`, which it's based on) primarily uses the **Kubernetes Python client library**. This client library, by default, will look for a `kubeconfig` file to connect to the cluster. This is the most common and recommended way to manage OpenShift resources with Ansible.

#### Option A: Using your existing `kubeconfig` (Recommended for most users)

If you regularly use `oc` (OpenShift CLI) or `kubectl` to interact with your cluster, you likely already have a `kubeconfig` file. By default, Ansible's `k8s` module will look for this file at `~/.kube/config`.

**Steps:**

1.  **Log in to your OpenShift cluster using `oc`:**

    ```bash
    oc login --token=<your_token> --server=<your_openshift_api_url>
    ```

    Or if you're using a username/password:

    ```bash
    oc login -u <username> -p <password> --server=<your_openshift_api_url>
    ```

    This command will authenticate you and typically update or create your `~/.kube/config` file with the necessary credentials and cluster information.

2.  **Verify your `kubeconfig`:**
    You can check the active context and cluster information with:

    ```bash
    kubectl config current-context
    kubectl config view
    ```

3.  **Ensure Python dependencies are met:**
    The `redhat.openshift.k8s` module relies on the Kubernetes Python client. Install it on your Ansible control node:

    ```bash
    pip install kubernetes openshift
    ```

    (Note: `openshift` package is for the OpenShift client, `kubernetes` for the generic Kubernetes client. Both are good to have.)

4.  **No extra parameters in playbook:**
    Since the module defaults to using `~/.kube/config`, your playbook doesn't need explicit connection parameters for the `k8s` module. The `hosts: localhost` and `connection: local` in your playbook are correct, as the Ansible control node itself is making the API calls, not connecting via SSH to the cluster nodes.

#### Option B: Specifying `kubeconfig` path or connection details directly in the playbook/inventory

If your `kubeconfig` is not in the default location, or you prefer to explicitly define connection parameters, you can do so:

  * **Specify `kubeconfig` path:**

    ```yaml
    - name: Create OpenShift resources
      hosts: localhost
      connection: local
      gather_facts: false
      tasks:
        - name: Create 'databases' namespace
          redhat.openshift.k8s:
            state: present
            kubeconfig: /path/to/your/custom/kubeconfig.yaml # Specify path
            # context: my-cluster-context # If you have multiple contexts in the file
            api_version: v1
            kind: Namespace
            name: databases
    ```

  * **Specify host and token/certs:**
    You can also provide the API server host, a token, or client certificates directly to the module. This is less common for general use but useful in specific scenarios (e.g., CI/CD pipelines where a service account token is provided).

    ```yaml
    - name: Create OpenShift resources
      hosts: localhost
      connection: local
      gather_facts: false
      tasks:
        - name: Create 'databases' namespace
          redhat.openshift.k8s:
            state: present
            host: https://api.yourcluster.example.com:6443 # Your OpenShift API URL
            api_key: <your_bearer_token> # Bearer token (e.g., from a service account)
            # ca_cert: /path/to/ca.crt # Optional: if your cluster uses a custom CA
            # validate_certs: false # Use with caution: skips SSL certificate validation
            api_version: v1
            kind: Namespace
            name: databases
    ```

    **How to get a bearer token (e.g., for a ServiceAccount):**

    1.  Create a ServiceAccount: `oc create sa ansible-runner -n <your-namespace>`
    2.  Grant it permissions (e.g., `edit` role for the namespace):
        `oc adm policy add-role-to-user edit -z ansible-runner -n <your-namespace>`
    3.  Get the token:
        ```bash
        oc get secret $(oc get sa ansible-runner -n <your-namespace> -o jsonpath='{.secrets[0].name}') -n <your-namespace> -o jsonpath='{.data.token}' | base64 --decode
        ```
        **Important:** Store tokens securely, ideally using Ansible Vault, not directly in playbooks.

## Create OpenShift Resource using Ansible

### 1\. Define the Playbook

As an example, we'll use the `redhat.openshift.k8s` module to create a namespace named "databases", a secret named "creds" containing the `MYSQL_ROOT_PASSWORD="rootpass"` key value, and a StatefulSet named "wp-db" using docker.io/mysql image and having the `MYSQL_ROOT_PASSWORD` configured from the "creds" secret.

- name: Create OpenShift resources for WordPress database
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Create 'databases' namespace
      redhat.openshift.k8s:
        state: present
        api_version: v1
        kind: Namespace
        name: databases

    - name: Create 'creds' secret in 'databases' namespace
      redhat.openshift.k8s:
        state: present
        api_version: v1
        kind: Secret
        namespace: databases
        name: creds
        type: Opaque
        data:
          MYSQL_ROOT_PASSWORD: "cm9vdHBhc3M=" # Base64 encoded "rootpass"

    - name: Create 'wp-db' StatefulSet in 'databases' namespace
      redhat.openshift.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: StatefulSet
          metadata:
            namespace: databases
            name: wp-db
          spec:
            selector:
              matchLabels:
                app: mysql
            serviceName: mysql
            replicas: 1
            template:
              metadata:
                labels:
                  app: mysql
              spec:
                containers:
                  - name: mysql
                    image: docker.io/mysql
                    ports:
                      - containerPort: 3306
                        name: mysql
                    env:
                      - name: MYSQL_ROOT_PASSWORD
                        valueFrom:
                          secretKeyRef:
                            name: creds
                            key: MYSQL_ROOT_PASSWORD
                    volumeMounts:
                      - name: mysql-persistent-storage
                        mountPath: /var/lib/mysql
            volumeClaimTemplates:
              - metadata:
                  name: mysql-persistent-storage
                spec:
                  accessModes: [ "ReadWriteOnce" ]
                  resources:
                    requests:
                      storage: 5Gi


### 2\. Run the Playbook

Once your Ansible environment is set up (collections installed, Python dependencies, and `kubeconfig` configured or explicit connection parameters provided), and the Ansible playbook defined, you can run your playbook:

```bash
ansible-playbook your_playbook_name.yml
```

**Example Output:**

```
PLAY [Create OpenShift resources for WordPress database] ***********************

TASK [Create 'databases' namespace] ********************************************
changed: [localhost]

TASK [Create 'creds' secret in 'databases' namespace] **************************
changed: [localhost]

TASK [Create 'wp-db' StatefulSet in 'databases' namespace] *********************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

After running, you can verify the resources on your OpenShift cluster using `oc` commands:

```bash
oc get ns databases
oc get secret creds -n databases
oc get statefulset wp-db -n databases
oc describe statefulset wp-db -n databases
```

This will confirm that Ansible successfully connected and created the specified resources.
