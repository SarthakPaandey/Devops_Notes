# Day 21: Infrastructure as Code with Terraform
**Date:** 9th January

## 1. What is IaC?
Infrastructure as Code manages and provisions infrastructure through code instead of manual processes.
*   **Benefits:** Consistency (No snowflake servers), Version Control (Git), Speed, Reusability.
*   **Tools:** Terraform (Cloud Agnostic), Ansible (Config Mgmt - Post Provisioning), Pulumi (Code-based).

## 2. Terraform Core Concepts
1.  **Provider:** A plugin that talks to APIs (AWS, Azure, Kubernetes).
2.  **Resource:** The object you want to create (EC2 instance, S3 Bucket).
3.  **Data Source:** A read-only lookup (Find the latest Ubuntu AMI ID).
4.  **State File (`terraform.tfstate`):** A JSON database tracking the mapping between your code and real-world resources.
    *   **Locking:** Essential for teams. Prevents two people from applying at once. Requires DynamoDB (AWS) or Consul.

---

## 3. Writing Logic (HCL - HashiCorp Configuration Language)

### Variables (`variables.tf`)
Define inputs.
```hcl
variable "region" {
  type        = string
  description = "AWS Region"
  default     = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "tags" {
  type = map(string)
  default = {
    Environment = "Dev"
    Project     = "Demo"
  }
}
```

### Main Logic (`main.tf`)
Use variables based on syntax: `var.variable_name`.
```hcl
provider "aws" {
  region = var.region
}

# Use Data Source to get latest AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id # Reference Data Source
  instance_type = "t2.micro"

  tags = var.tags
}
```

### Outputs (`outputs.tf`)
Print values after apply (e.g., The Public IP).
```hcl
output "instance_ip" {
  value = aws_instance.web[*].public_ip
}
```

---

## 4. The Lifecycle Workflow
1.  `terraform init`: Analysis the code, downloads providers.
2.  `terraform validate`: Checks syntax errors.
3.  `terraform plan`: Dry Run. Shows what *will* happen (+ Create, - Destroy, ~ Change).
    *   **ALWAYS READ THE PLAN.**
4.  `terraform apply`: Executes the plan. Updates State file.
    *   `terraform apply -auto-approve`: Skip "yes" prompt (CI/CD).
5.  `terraform destroy`: Deletes everything managed by the state file.

---

## 5. State Management Deep Dive
The most critical file.
*   **Local State:** Stored on your laptop. BAD for teams.
*   **Remote State:** Stored on S3/GCS. Good for teams.
*   **State Locking:** Prevents corruption. Creating a `DynamoDB` table with `LockID` allows Terraform to "acquire lock" during apply.

**Debugging State:**
*   `terraform state list`: Show all resources.
*   `terraform state show <resource>`: Show details.
*   `terraform state rm <resource>`: Stop managing a resource (doesn't delete it).
*   `terraform import <resource> <id>`: Bring an existing (manual) resource into Terraform management.

---

## 6. Interview Questions
1.  **Q: What happens if I delete a resource manually in AWS console?**
    *   *A:* Terraform detects "Drift". When you run `terraform plan`, it sees the resource is missing in reality but present in Code. It proposes to **Create** it again (Self-Healing).
2.  **Q: Difference between `count` and `for_each`?**
    *   *A:* **Count:** Creates a list (`web[0]`, `web[1]`). If you remove index 0, index 1 becomes index 0 (renaming/destroying). **For_Each:** Creates a map (`web["server-a"]`). Stable keys. Better for lists that change.
3.  **Q: How do I debug Terraform?**
    *   *A:* `export TF_LOG=DEBUG`. Run `terraform console` to test interpolation logic (`local.name`).
    *   Use `terraform fmt` to fix indentation automatically.
