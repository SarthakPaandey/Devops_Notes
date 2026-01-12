# Day 22: Advanced Terraform - Modules, State & Best Practices
**Date:** 12th January

## 1. Remote State Management
Storing `terraform.tfstate` on your laptop is bad practice.
*   **Risk:** Laptop stolen/crashed = Lost infrastructure map. No collaboration possible.
*   **Solution:** Store state in a shared, locked backend.

### AWS S3 Backend Example
```hcl
terraform {
  backend "s3" {
    bucket         = "tf-state-prod-12345"
    key            = "networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks" # For Locking
    encrypt        = true
  }
}
```
*   **Locking:** DynamoDB prevents two developers (`alice` and `bob`) from running `terraform apply` at the same exact second, which would corrupt the state file.
*   **Encryption:** S3 Server Side Encryption protects sensitive data in state (like DB passwords).

---

## 2. Terraform Modules (Don't Repeat Yourself)
Modules allow you to package resources into reusable components.
*   **Module Source:** Can be Local Path, Git URL, or Terraform Registry.

### Using a Module
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

### Creating a Custom Module
Folder Structure:
```text
modules/
  webserver/
    main.tf      (logic)
    variables.tf (inputs)
    outputs.tf   (return values)
```

**Calling Custom Module:**
```hcl
module "web" {
  source = "./modules/webserver"
  instance_type = "t3.medium"
}
```

---

## 3. Workspaces (Environment Isolation)
Manage multiple environments (dev, staging, prod) with the **same code** but **different state files**.
*   `terraform workspace new dev`
*   `terraform workspace select dev`
*   `terraform apply` (Creates state: `terraform.tfstate.d/dev`)

**Code Usage:**
```hcl
resource "aws_instance" "app" {
  count = terraform.workspace == "prod" ? 3 : 1
}
```
*   **Pros:** Quick switching.
*   **Cons:** easy to accidentally apply to Prod. **Best Practice:** Use separate directories/repos for Prod.

---

## 4. Provisioners (The "Last Resort")
Sometimes you need to run a script on the machine or locally *after* creation.
**Warning:** Only use provisioners if there is no other way (like UserData or packer).

### `local-exec` (Run on your laptop)
```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> private_ips.txt"
  }
}
```

### `remote-exec` (Run on the server)
Requires SSH connection.
```hcl
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }
```

### `null_resource`
A dummy resource used to trigger provisioners when nothing else changes.
```hcl
resource "null_resource" "cluster" {
  triggers = {
    cluster_instance_ids = join(",", aws_instance.cluster.*.id)
  }
  provisioner "local-exec" {
    command = "./configure-cluster.sh"
  }
}
```

---

## 5. Advanced Functions & Loops
*   **Lookup:** `lookup(map, key, default)`
*   **Element:** `element(list, index)`
*   **File:** `file("script.sh")` reads content.
*   **Templatefile:** Renders a template with variables.
    ```hcl
    user_data = templatefile("init.sh.tpl", {
      package_name = var.package
    })
    ```

### Dynamic Blocks
Loop inside a resource block (e.g., Security Group Rules).
```hcl
resource "aws_security_group" "sg" {
  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port = ingress.value
      to_port   = ingress.value
      protocol  = "tcp"
    }
  }
}
```

---

## 6. Interview Questions
1.  **Q: How do you handle secrets in Terraform?**
    *   *A:* **Never** hardcode them. Use Environment Variables (`TF_VAR_db_password`), AWS Secrets Manager Data Sources, or Vault Provider. State file *will* contain secrets in plain text, so encrypt the backend (S3).
2.  **Q: What is `terraform taint`?**
    *   *A:* Marks a resource as "degraded" or "needs replacement". The next Apply will destroy and recreate it. Useful if a provisioner failed but the resource creation succeeded.
3.  **Q: Dependencies between modules?**
    *   *A:* If Module B needs Output from Module A, pass `module.A.output_value` as an input variable to Module B. Terraform builds a dependency graph and runs Apply in order.
