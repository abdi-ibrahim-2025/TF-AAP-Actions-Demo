# Ansible Automation Platform / HCP Terraform Demo

This repository demonstrates the integration between Ansible Automation Platform (AAP) and HashiCorp Cloud Platform (HCP) Terraform, showcasing automated provisioning and configuration of AWS EC2 instances using Terraform Actions and AAP Event-Driven Ansible (EDA).

## Overview

This demo uses Terraform to provision AWS infrastructure and automatically trigger AAP workflows through EDA Event Streams. When EC2 instances are created and added to the AAP inventory, Terraform actions send events to AAP, which then executes playbooks to configure the newly provisioned infrastructure.

## Product Requirements

Before starting, ensure you have access to the following:

- **Ansible Automation Platform (AAP)**: Set up an AAP sandbox using the [30-day trial](https://developers.redhat.com/developer-sandbox)
- **HCP Terraform**: Active account with workspace creation permissions
- **AWS Account**: With permissions to create EC2 instances, security groups, and key pairs
- **GitHub Repository**: Fork this repository to your own account for customization ([original repo](https://github.com/HichamMourad/TF-AAP-Actions-Demo/tree/main))

> **Note**: This repository is maintained by Hicham Mourad, Red Hat Technical Marketing. You should fork the repository if you're creating your own demo to maintain independent control.

## Phase 1: AWS and AAP Setup (Prerequisites)

This phase configures the external systems that Terraform will interact with via the AAP API and the EDA Event Stream.

### 1. AWS Key Pair & AAP Machine Credential

#### AWS Configuration
1. Navigate to the AWS Console in your target region (e.g., `us-east-1`)
2. Go to **EC2 → Key Pairs**
3. Create a new EC2 Key Pair with the name: `tf-aap-demo-key` (or the name specified in your `ssh_key_name` variable)
4. Download and save the private key file (`.pem`)

#### AAP Configuration
1. Log in to your AAP instance
2. Navigate to **Resources → Credentials**
3. Click **Add** to create a new credential
4. Configure the credential:
   - **Name**: `TF Demo AWS Key`
   - **Credential Type**: `Machine`
   - **Username**: `ec2-user` (to match RHEL/Amazon Linux AMI)
   - **SSH Private Key**: Paste the entire private key content from your downloaded `.pem` file
5. Save the credential

### 2. Configure AAP Workflows & Job Templates

Ensure the following Job Templates/Workflows exist in AAP under the **Default** Organization:

1. **New AWS Provisioning Workflow**
   - Triggered after EC2 instances are created
   - Should include tasks for initial provisioning and configuration

2. **Update AWS Provisioning Job**
   - Triggered after hosts are added to the inventory
   - Should include tasks for updating and configuring existing infrastructure

> **Important**: The names must match exactly as they are referenced in the Terraform configuration.

### 3. Create and Configure EDA Event Stream

#### Create Event Stream
1. In AAP, navigate to **Event-Driven Ansible → Event Streams**
2. Click **Create event stream**
3. Configure the event stream:
   - **Name**: `TF Actions Event Stream`
   - Note other settings as needed for your environment
4. **Save** the event stream
5. **Record the exact URL** of this Event Stream - you will need it for HCP Terraform configuration

#### Create EDA Credential
1. Navigate to **Resources → Credentials**
2. Create a new credential for Event Stream authentication:
   - **Name**: `TF Event Stream Credential`
   - **Credential Type**: Select appropriate type for basic authentication
   - **Username**: Choose a username (e.g., `tf-es-user`)
   - **Password**: Choose a secure password
3. Save these credentials - you'll need to provide them as Terraform variables (`tf-es-username` and `tf-es-password`)

## Phase 2: HCP Terraform Workspace Configuration

### 1. Create New Workspace

1. Log in to [HCP Terraform](https://app.terraform.io)
2. Select your organization
3. Click **New → Workspace**
4. Choose **Version control workflow**
5. Connect to your forked Git repository
6. Configure workspace settings:
   - **Workspace Name**: Choose a descriptive name (e.g., `aap-terraform-demo`)
   - **Terraform Working Directory**: `terraform`
7. Create the workspace

### 2. Set Workspace Variables

Navigate to your workspace's **Variables** section and configure the following variables:

#### Environment Variables

| Key | Value | Category | Sensitive | Purpose |
|-----|-------|----------|-----------|---------|
| `AWS_ACCESS_KEY_ID` | Your AWS Access Key ID | Environment | ✓ Yes | AWS API Authentication |
| `AWS_SECRET_ACCESS_KEY` | Your AWS Secret Access Key | Environment | ✓ Yes | AWS API Authentication |
| `aap_host` | Your AAP URL (e.g., `https://your-aap-instance.com`) | Environment | No | AAP API Host URL |
| `aap_username` | Your AAP Username | Environment | No | AAP API Authentication (for Terraform Provider) |
| `aap_password` | Your AAP Password | Environment | ✓ Yes | AAP API Authentication |

#### Terraform Variables

| Key | Value | Category | Sensitive | Purpose |
|-----|-------|----------|-----------|---------|
| `ssh_key_name` | `tf-aap-demo-key` | Terraform | No | Name of the AWS Key Pair |
| `aap_eventstream_url` | The EDA Event Stream URL | Terraform | No | Target URL for the `aap_eda_eventstream_post` actions |
| `tf-es-username` | Your EDA Credential Username | Terraform | ✓ Yes | Username for EDA Stream Authentication |
| `tf-es-password` | Your EDA Credential Password | Terraform | ✓ Yes | Password for EDA Stream Authentication |

> **Important**: Ensure sensitive variables are marked as sensitive to protect credentials.

### 3. Configure Terraform Resources

1. Review the `terraform/main.tf` file and adjust the `count` parameter in the `aws_instance` resource to control the number of instances to create (default is `0` - update as needed)
2. Verify the AMI ID matches your target region
3. Commit and push any changes to trigger a Terraform plan

## Repository Structure

```
.
├── README.md
├── terraform/
│   └── main.tf                           # Main Terraform configuration
└── playbooks/
    ├── configure_firewalld-rhel.yml      # Firewall configuration for RHEL
    ├── install_nginx-rhel.yml            # NGINX installation for RHEL
    └── install_nginx-debian.yml          # NGINX installation for Debian
```

## How It Works

1. **Terraform Provision**: Terraform provisions AWS EC2 instances based on the configuration in `main.tf`
2. **Action Trigger (Create)**: After instance creation, the `after_create` lifecycle event triggers the `aap_eda_eventstream_post.create` action
3. **Event Stream**: The action posts an event to the AAP EDA Event Stream
4. **EDA Processing**: EDA processes the event and triggers the "New AWS Provisioning Workflow"
5. **Host Addition**: Terraform adds the new instances to the AAP inventory
6. **Action Trigger (Update)**: After host addition, another `after_create` event triggers the `aap_eda_eventstream_post.update` action
7. **Configuration**: EDA triggers the "Update AWS Provisioning Job" to configure the instances

## Running the Demo

1. Ensure all prerequisites are configured (Phase 1)
2. Configure your HCP Terraform workspace (Phase 2)
3. Update the `count` parameter in `aws_instance.web_server` to the desired number of instances
4. Commit and push changes to trigger Terraform
5. Monitor the Terraform run in HCP Terraform
6. Observe the triggered AAP workflows in the AAP UI

## Additional Notes

- The demo uses **Red Hat Enterprise Linux 9 (HVM)** as the default AMI
- Security groups allow SSH (port 22) and HTTP (port 80) traffic
- The `ansible_user` is set to `ec2-user` to match the RHEL/Amazon Linux AMI
- Terraform requires version `~> v1.14.0` and uses AAP provider version `1.4.0-devpreview1`
- All AAP resources must be in the "Default" organization

## Troubleshooting

- **Event Stream not receiving events**: Verify the `aap_eventstream_url` is correct and credentials are valid
- **Job templates not found**: Ensure job template names exactly match those in the Terraform configuration
- **SSH connection failures**: Verify the Machine Credential has the correct private key and username (`ec2-user`)
- **AWS authentication issues**: Check that AWS credentials are properly set in the workspace environment variables

## Credits

Original repository and demo created by **Hicham Mourad**, Red Hat Technical Marketing.
