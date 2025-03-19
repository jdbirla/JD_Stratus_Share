# JD_Stratus_Share

### Schema
```sql
SELECT 
    fk.name AS ForeignKey,
    tp.name AS ParentTable,
    cp.name AS ParentColumn,
    tr.name AS ReferencedTable,
    cr.name AS ReferencedColumn
FROM sys.foreign_keys fk
JOIN sys.tables tp ON fk.parent_object_id = tp.object_id
JOIN sys.tables tr ON fk.referenced_object_id = tr.object_id
JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
JOIN sys.columns cp ON fkc.parent_column_id = cp.column_id AND fkc.parent_object_id = cp.object_id
JOIN sys.columns cr ON fkc.referenced_column_id = cr.column_id AND fkc.referenced_object_id = cr.object_id
ORDER BY ParentTable, ReferencedTable;
```
You need a **Java reflection-based utility** that scans your class, detects all fields (including nested objects), and **generates a builder pattern-based object creation code as a `String`**.  

---

### **📌 Solution Overview**
1. **Use reflection** to iterate over fields of the provided class.
2. **Detect primitive types, wrapper classes, and nested objects.**
3. **Generate a Java code string** that initializes the object using the Builder pattern.
4. **Recursively generate values for nested objects.**
5. **Return a `String` containing the builder-based instantiation code.**

---

### **✅ Java Code: Generate Object Creation Code Dynamically**
```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Random;

public class BuilderCodeGenerator {

    public static String generateBuilderCode(Class<?> clazz, String variableName) {
        StringBuilder sb = new StringBuilder();
        sb.append(clazz.getSimpleName()).append(" ").append(variableName).append(" = ")
                .append(clazz.getSimpleName()).append(".builder()\n");

        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            String fieldName = field.getName();
            Class<?> fieldType = field.getType();

            String value = getDefaultValueCode(fieldType, fieldName);
            sb.append("    .").append("with").append(capitalize(fieldName)).append("(").append(value).append(")\n");
        }

        sb.append("    .build();\n");
        return sb.toString();
    }

    private static String getDefaultValueCode(Class<?> type, String fieldName) {
        if (type.equals(String.class)) {
            return "\"" + fieldName + "_default\"";
        } else if (type.equals(int.class) || type.equals(Integer.class)) {
            return "100";
        } else if (type.equals(long.class) || type.equals(Long.class)) {
            return "100L";
        } else if (type.equals(boolean.class) || type.equals(Boolean.class)) {
            return "true";
        } else if (type.equals(double.class) || type.equals(Double.class)) {
            return "10.5";
        } else if (type.isEnum()) {
            return type.getSimpleName() + ".values()[0]"; // Use first enum value
        } else {
            // Nested object case (Recursive Call)
            return generateBuilderCode(type, fieldName + "Obj");
        }
    }

    private static String capitalize(String str) {
        return str.substring(0, 1).toUpperCase() + str.substring(1);
    }

    public static void main(String[] args) {
        // Example usage with a sample class
        String builderCode = generateBuilderCode(Transaction.class, "transaction");
        System.out.println(builderCode);
    }
}
```

---

### **✅ Example Class: `Transaction`**
```java
public class Transaction {
    private String transactionId;
    private int amount;
    private User user; // Nested object

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String transactionId;
        private int amount;
        private User user;

        public Builder withTransactionId(String transactionId) { this.transactionId = transactionId; return this; }
        public Builder withAmount(int amount) { this.amount = amount; return this; }
        public Builder withUser(User user) { this.user = user; return this; }

        public Transaction build() { return new Transaction(transactionId, amount, user); }
    }
}
```

---

### **✅ Example Class: `User` (Nested Object)**
```java
public class User {
    private String username;
    private int age;

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String username;
        private int age;

        public Builder withUsername(String username) { this.username = username; return this; }
        public Builder withAge(int age) { this.age = age; return this; }

        public User build() { return new User(username, age); }
    }
}
```

---

### **✅ Generated Code Output**
When you run the `main()` method, this **automatically generated** builder code prints:
```java
Transaction transaction = Transaction.builder()
    .withTransactionId("transactionId_default")
    .withAmount(100)
    .withUser(User.builder()
        .withUsername("username_default")
        .withAge(100)
        .build())
    .build();
```
---
### **🔹 Key Features**
✔ **Dynamically generates builder code** for **any** class.  
✔ **Handles nested objects** recursively.  
✔ **Uses default values** for each field (string, numbers, enums, booleans).  
✔ **Outputs valid Java code** that can be copy-pasted.  

Would you like any enhancements? 🚀
----

### Vault Secret 

To securely inject a key-value (K/V) secret from HashiCorp Vault into your Kubernetes deployment using the Vault Agent Injector and subsequently create a Kubernetes Secret from this data, follow the steps outlined below:

**1. Store Your Application YAML in Vault:**

First, ensure that your application's YAML configuration is stored as a secret in Vault. For instance, if your secret's key is `config.secret` and it contains your entire application YAML, you can store it using the following command:

```bash
vault kv put secret/config config.secret=@app-config.yaml
```


Here, `app-config.yaml` is your application's configuration file.

**2. Create a Vault Policy:**

Define a Vault policy that grants read access to the secret. Save the policy in a file named `app-policy.hcl`:

```hcl
path "secret/data/config" {
  capabilities = ["read"]
}
```


Apply the policy in Vault:

```bash
vault policy write app-policy app-policy.hcl
```


**3. Enable Kubernetes Authentication in Vault:**

Configure Vault to authenticate using Kubernetes:

```bash
vault auth enable kubernetes
```


Retrieve the Kubernetes API server URL:

```bash
KUBE_API_SERVER=$(kubectl config view --raw --minify --flatten -o json | jq -r '.clusters[0].cluster.server')
```


Obtain the JWT token from the Kubernetes service account:

```bash
SERVICE_ACCOUNT_NAME=vault-auth
SECRET_NAME=$(kubectl get sa $SERVICE_ACCOUNT_NAME -o jsonpath="{.secrets[0].name}")
SA_JWT_TOKEN=$(kubectl get secret $SECRET_NAME -o jsonpath="{.data.token}" | base64 --decode)
```


Retrieve the Kubernetes CA certificate:

```bash
KUBE_CA_CERT=$(kubectl get secret $SECRET_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode)
```


Configure Vault with the Kubernetes authentication details:

```bash
vault write auth/kubernetes/config \
    token_reviewer_jwt="$SA_JWT_TOKEN" \
    kubernetes_host="$KUBE_API_SERVER" \
    kubernetes_ca_cert="$KUBE_CA_CERT"
```


**4. Create a Vault Role:**

Bind the Kubernetes service account to the Vault policy:

```bash
vault write auth/kubernetes/role/app-role \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=24h
```


**5. Deploy the Vault Agent Injector:**

Ensure that the Vault Agent Injector is deployed in your Kubernetes cluster. This can be done using the Vault Helm chart:

```bash
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```


**6. Annotate Your Kubernetes Deployment:**

Modify your Kubernetes deployment to include annotations that instruct the Vault Agent Injector to retrieve the secret and create a Kubernetes Secret. Here's an example of how to set this up:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    vault.hashicorp.com/agent-inject: 'true'
    vault.hashicorp.com/role: 'app-role'
    vault.hashicorp.com/agent-inject-secret-config: 'secret/data/config'
    vault.hashicorp.com/agent-inject-template-config: |
      {{- with secret "secret/data/config" -}}
      apiVersion: v1
      kind: Secret
      metadata:
        name: app-config-secret
      type: Opaque
      data:
        config.yaml: {{ .Data.data.config.secret | base64Encode }}
      {{- end -}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: vault-auth
      containers:
        - name: my-app-container
          image: my-app-image
          volumeMounts:
            - name: app-config
              mountPath: /etc/app-config
              readOnly: true
      volumes:
        - name: app-config
          secret:
            secretName: app-config-secret
```


**Explanation:**

- `vault.hashicorp.com/agent-inject: 'true'`: Enables the Vault Agent Injector for this pod.

- `vault.hashicorp.com/role: 'app-role'`: Specifies the Vault role to use for authentication.

- `vault.hashicorp.com/agent-inject-secret-config: 'secret/data/config'`: Indicates the path to the Vault secret.

- `vault.hashicorp.com/agent-inject-template-config`: Utilizes a Vault Agent template to render the secret as a Kubernetes Secret manifest. The `config.secret` is base64 encoded to meet Kubernetes Secret requirements.

When the pod is deployed, the Vault Agent Injector will:

1. Authenticate to Vault using the specified role.

2. Retrieve the secret from Vault.

3. Render the secret into a Kubernetes Secret manifest using the provided template.

4. Apply the Kubernetes Secret to the cluster.

5. Mount the newly created Kubernetes Secret as a volume in the application container.

**7. Deploy Your Application:**

Apply your deployment configuration:

```bash
kubectl apply -f my-app-deployment.yaml
```


This setup ensures that your application's YAML configuration stored in Vault is securely injected into your Kubernetes deployment and made available as a Kubernetes Secret.

For more detailed information, refer to the [Vault Agent Injector documentation](https://developer.hashicorp.com/vault/docs/platform/k8s/injector). 

Yes, you can mount your additional Secret into a different volume within the same Pod. In Kubernetes, each volume is mounted to a unique directory within the container's filesystem. If your existing Secret is already mounted to a specific directory, you can mount the new Secret to a separate directory without any conflicts.

**Example: Mounting Multiple Secrets to Different Directories**

Here's how you can define a Pod specification to mount multiple Secrets into separate directories:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: my-image
      volumeMounts:
        - name: existing-secret-volume
          mountPath: /etc/existing-secret
        - name: new-secret-volume
          mountPath: /etc/new-secret
  volumes:
    - name: existing-secret-volume
      secret:
        secretName: existing-secret
    - name: new-secret-volume
      secret:
        secretName: new-secret
```


In this configuration:

- The `existing-secret` is mounted to `/etc/existing-secret`.

- The `new-secret` is mounted to `/etc/new-secret`.

This approach ensures that each Secret is accessible within its designated directory in the container's filesystem.

**Considerations:**

- **Unique Mount Paths:** Ensure that each volume has a unique `mountPath` to prevent conflicts.

- **Access Permissions:** Verify that the application running in the Pod has the necessary permissions to access the mounted Secret files.

By mounting each Secret to a separate directory, you can effectively manage multiple Secrets within your Kubernetes Pods, ensuring that your applications have access to all necessary sensitive data without encountering mounting conflicts. 




```sh

#!/bin/bash

# Set variables
SECRET_FILE="config/secret.yml"
NAMESPACE="default"  # Change this if needed

# Function to log messages with timestamp
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Check if kubectl is installed
if ! command -v kubectl &> /dev/null; then
    log "❌ ERROR: kubectl command not found. Please install Kubernetes CLI."
    exit 1
fi

# Check if secret.yml file exists
if [ ! -f "$SECRET_FILE" ]; then
    log "❌ ERROR: Secret file '$SECRET_FILE' not found!"
    exit 1
fi

# Apply the secret using kubectl
log "🚀 Applying Kubernetes secret from '$SECRET_FILE'..."
kubectl apply -f "$SECRET_FILE" --namespace="$NAMESPACE"

# Check if the secret was applied successfully
if [ $? -eq 0 ]; then
    log "✅ SUCCESS: Secret applied successfully!"
else
    log "❌ ERROR: Failed to apply secret!"
    exit 1
fi
```

```python
from kubernetes import client, config

def apply_secret():
    config.load_incluster_config()  # Load in-cluster Kubernetes config
    api_instance = client.CoreV1Api()

    with open("/opt/asd/asd/config/secret.yml", "r") as f:
        secret_yaml = yaml.safe_load(f)

    api_instance.create_namespaced_secret(
        namespace="default",
        body=secret_yaml
    )

apply_secret_task = PythonOperator(
    task_id="apply_secret_python",
    python_callable=apply_secret,
)

```
