# Vault Integration 

Here is a complete and well-structured **README.md** file for your presentation on **Vault integration in Kubernetes using HashiCorp Vault Injector Annotations**. It explains all the basic concepts, the problem with manual secrets, the role of Vault, different secret management tools, and how to use annotations for Vault Agent Injector integration.

---

# 🔐 Vault Integration in Kubernetes Using HashiCorp Vault Injector

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Basics of Secrets in Kubernetes](#basics-of-secrets-in-kubernetes)
3. [Problems with Manually Applied Kubernetes Secrets](#problems-with-manually-applied-kubernetes-secrets)
4. [What is Vault?](#what-is-vault)
5. [Vault Providers](#vault-providers)
6. [HashiCorp Vault + Kubernetes Integration](#hashicorp-vault--kubernetes-integration)
7. [What is Vault Agent Injector?](#what-is-vault-agent-injector)
8. [Kubernetes Annotations for Vault Injector](#kubernetes-annotations-for-vault-injector)
9. [Complete Example](#complete-example)
10. [Security Best Practices](#security-best-practices)
11. [Conclusion](#conclusion)

---

## 🔹 Introduction

This guide explains how to securely inject secrets from **HashiCorp Vault** into applications running on **Kubernetes** using the **Vault Agent Injector**, leveraging annotations for secret injection without modifying application code.

---

## 🔐 Basics of Secrets in Kubernetes

Kubernetes provides a native way to store and manage sensitive data using [`Secret`](https://kubernetes.io/docs/concepts/configuration/secret/) resources.

### Examples:

* API keys
* Passwords
* TLS certificates

### Ways to use Secrets:

* As environment variables
* As mounted volumes (files)
* Direct access using `kubectl`

---

## ⚠️ Problems with Manually Applied Kubernetes Secrets

While Kubernetes Secrets are base64-encoded and stored in etcd, they have **several limitations**:

| Issue              | Description                                                        |
| ------------------ | ------------------------------------------------------------------ |
| 🔓 Weak Encryption | Stored as base64, not encrypted by default in etcd.                |
| 🔁 Manual Rotation | Secrets like tokens or passwords must be updated manually.         |
| 👥 RBAC Misuse     | Improper RBAC can expose secrets to unauthorized users.            |
| 📦 CI/CD Exposure  | Secrets committed or exposed in Git repos or pipelines by mistake. |

---

## 🛡️ What is Vault?

[HashiCorp Vault](https://www.vaultproject.io/) is a **secrets management tool** designed to securely store, access, and manage sensitive information like tokens, passwords, and certificates.

### Key Features:

* Dynamic Secrets
* Automatic Secret Rotation
* Encryption as a Service
* Audit Logging
* Fine-Grained Access Control (via Policies)

---

## 🔄 Vault Providers

Vault alternatives (providers of secret management):

| Tool                     | Description                           |
| ------------------------ | ------------------------------------- |
| 🔐 **HashiCorp Vault**   | Advanced, extensible, dynamic secrets |
| ☁️ AWS Secrets Manager   | AWS-specific secret store             |
| ☁️ Azure Key Vault       | Microsoft Azure’s secret store        |
| ☁️ Google Secret Manager | Google Cloud's secure store           |
| 🧪 CyberArk              | Enterprise-level secrets management   |
| 🧩 SOPS (by Mozilla)     | GitOps-friendly secret encryption     |

---

## 🔗 HashiCorp Vault + Kubernetes Integration

Vault integrates with Kubernetes using the following methods:

1. **Vault Agent Sidecar Injector** (focus of this guide)
2. CSI Driver
3. Vault SDK in the application

---

## 🤖 What is Vault Agent Injector?

Vault Agent Injector is a **Kubernetes Mutating Admission Webhook** that:

1. Watches for specific annotations on Pods.
2. Mutates the Pod spec at creation time.
3. Injects:

   * A Vault Agent sidecar container
   * An `init` container (optional)
   * Volume mounts containing secrets
4. Vault Agent retrieves secrets and writes them to shared volumes or files.

---

## 🧾 Kubernetes Annotations for Vault Injector

Vault annotations define how the Vault Agent behaves per pod.

### Required Annotations:

```yaml
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/role: "my-k8s-role"
vault.hashicorp.com/agent-inject-secret-<filename>: "path/to/secret"
```

### Common Annotations:

| Annotation                                         | Purpose                                |
| -------------------------------------------------- | -------------------------------------- |
| `vault.hashicorp.com/agent-inject`                 | Enable the Vault injector              |
| `vault.hashicorp.com/role`                         | Vault Kubernetes auth role             |
| `vault.hashicorp.com/agent-inject-secret-<name>`   | Vault secret path                      |
| `vault.hashicorp.com/agent-inject-template-<name>` | Custom template for secret             |
| `vault.hashicorp.com/agent-pre-populate-only`      | If `true`, agent exits after injection |

---

## 📦 Complete Example

### Step 1: Enable Kubernetes Auth in Vault

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
  token_reviewer_jwt="<SERVICE_ACCOUNT_JWT>" \
  kubernetes_host="<K8S_API_URL>" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Step 2: Create a Role in Vault

```bash
vault write auth/kubernetes/role/my-k8s-role \
    bound_service_account_names=vault-sa \
    bound_service_account_namespaces=default \
    policies=my-policy \
    ttl=24h
```

### Step 3: Store a Secret in Vault

```bash
vault kv put secret/app/config username="admin" password="s3cr3t"
```

### Step 4: Annotated Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-k8s-role"
    vault.hashicorp.com/agent-inject-secret-config: "secret/app/config"
spec:
  serviceAccountName: vault-sa
  containers:
    - name: myapp
      image: myapp:latest
      volumeMounts:
        - name: vault-secrets
          mountPath: /vault/secrets
  volumes:
    - name: vault-secrets
      emptyDir: {}
```

> After injection, Vault Agent creates a file at `/vault/secrets/config` containing the secret values.

---

## 🔐 Security Best Practices

* Use **dynamic secrets** where possible (e.g., DB creds).
* Implement strict **Vault policies** and `lease` durations.
* Enable **audit logging** in Vault.
* Avoid hardcoding secret paths—use templates.
* Use a dedicated **service account** per application or namespace.

---

## ✅ Conclusion

Using HashiCorp Vault with Kubernetes annotations and the Vault Agent Injector provides:

* Centralized and secure secret storage
* Automatic secret injection at pod runtime
* Better auditability and control
* No code change required

It solves the limitations of static Kubernetes secrets and enables you to implement **Zero Trust** principles in your containerized applications.

---

Let me know if you also need:

* A demo app or YAML manifests
* Vault setup Helm charts
* CI/CD integration steps with Jenkins/GitHub Actions

I can help you build a working example or create slides based on this README.
