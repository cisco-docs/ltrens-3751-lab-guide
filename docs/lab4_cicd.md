# Lab 4 - CI/CD Integration

In modern software development, you continuously build, test and deploy iterative code changes. This process helps reduce the chance of developing new code based on buggy or failed previous versions. These principles can also be applied to network infrastructure configuration. Creating a Continuous Integration / Continuous Development (CI/CD) process for network configuration reduces the amount of human intervention, and minimizes risk associated with changes.

Consider the limitations imposed in a production environment, by following the simple or comprehensive example. Storing the code locally allows only a single user to make modifications. Making the code available in a Git repository would not fully address the problem as the Terraform state is still stored locally. Only the user who ran `terraform apply` would be aware of the latest state of the configuration.

This lab should help you get started with creating a basic CI/CD environment using GitLab. It serves as an example. GitLab offers powerful features such as the ability to create Git repositories, create teams, manage Terraform state, and CI/CD.

!!! note
    Note that a plethora of alternative tooling is available such as Jenkins or GitHub. While this guide focuses on GitLab, the same principles apply. It is worthwhile to understand whether your organization already offers any of these solutions for you to leverage.

This guide helps you set up a pipeline with the following stages:

- Validate
- Plan
- Deploy
- Cleanup

## Step 1: Getting started with Gitlab

The UbuntuVM has `GitLab-CE Community Edition` and `GitLab Runner` pre-installed as Docker containers.

- **GitLab-CE Community Edition**: This is the self-managed version of GitLab, providing a platform for hosting repositories, managing pipelines, and facilitating DevOps workflows.
- **GitLab Runner**: A lightweight, build-executor application used to run CI/CD jobs from GitLab pipelines.

The GitLab Runner is already registered to the pre-installed GitLab-CE instance. This ensures that your pipelines are ready to execute jobs seamlessly without requiring additional configuration for runner registration.

A GitLab account has already been set up for you. Below are the credentials to access GitLab:

| Device Name | Management IP: | Username:	| Password: | Interface:|
|-------------|-----------------|-----------| ----------| -------|
| *GitLab* | 198.18.133.101 | ```root``` | ```C1sco12345``` | [web](https://198.18.133.101){ .md-button } |

!!! note
    Note that all GitLab features used in this guide are available in the free tier. For more information about features and pricing see: [GitLab Pricing](https://about.gitlab.com/pricing/)

!!! note
    You will be required to generate a new Token for GitLab access as it might have expired, causing the pipeline to fail. Follow the steps below to generate a new token:

**`GITLAB_TOKEN` Change Procedure**
  - From Gitlab home, click on `Groups` -> `labuser` 
  - On the left pane select `Settings` -> `Access Tokens`
  - There should be a token already created with the name `GITLAB_TOKEN`. Click the Refresh icon to regenerate the new token. Copy the new token as you will only see it once. 
  - Now, on the left pane, select `CI/CD` -> `Variables`
  - Locate the variable `GITLAB_TOKEN` and click the Edit icon to edit the variable.
  - Paste the new token in the `Value` field and click on `Save changes`

    Now, please continue with the lab and the CI/CD pipeline should now be working as expected. 

**Import repository**

Next, select `Projects` -> `Create a project` -> `Import Project`.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig1.png){ width="850" }
</figure>

Under `Import project from` select `Repository by URL` to import an existing Git repository. Set the `Git repository URL` to:

```cli
https://github.com/netascode/nac-ise-comprehensive-example.git
```

Give the project a `Project name`:

```cli
Nac Ise Comprehensive Example
```

Click on `Create project` once you are satisfied with your configuration.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig3.png){ width="800" }
</figure>

## Step 2: Creating a pipeline

In your newly created GitLab repository, click the `Code` button, then copy the URL displayed under `Clone with HTTPS` by clicking the `Copy URL` icon.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig4.png){ width="800" }
</figure>

Start "Visual Studio Code" and open a new terminal by selecting `Terminal -> New Terminal` from the menu.

!!! note
    Make sure you are not in the existing git repository, otherwise you will get an error. Use `cd ..` to go up one level.

In the terminal window type the following command to clone the repository:

```cli
git -c http.sslVerify=false clone https://198.18.133.101/sda_as_code/nac-ise-comprehensive-example.git
```

!!! note
    You might be prompted for username and password (root/C1sco12345)

```
PS C:\Users\admin\Desktop> git -c http.sslVerify=false clone https://198.18.133.101/sda_as_code/nac-ise-comprehensive-example.git
Cloning into 'nac-ise-comprehensive-example'...
remote: Enumerating objects: 23, done.
remote: Total 23 (delta 0), reused 0 (delta 0), pack-reused 23 (from 1)
Receiving objects: 100% (23/23), 4.07 KiB | 297.00 KiB/s, done.
Resolving deltas: 100% (9/9), done.
```

Then open the newly created folder in "Visual Studio Code".

<figure markdown>
  ![](./assets/vs_terminal2.png){ width="400" }
</figure>

On the Workspace Trust dialog, select **Yes, I trust the authors** to enable all features in the workspace.

<figure markdown>
  ![](./assets/vs_terminal3.png){ width="400" }
</figure>

!!! note
    Note that `git` command was executed with option `-c http.sslVerify=false` which is used to disable SSL certificate verification when making HTTPS requests. This means that Git will not check the validity Gitlab self-signed SSL certificate. You can enable the option globally for all future Git operations by using the following command:

    ```cli
    git config --global http.sslVerify false
    ```

## Step 3: Review the Existing .gitlab-ci.yml File

There is already a `.gitlab-ci.yml` file located in the root of this folder:

<figure markdown> ![](./assets/nac-ise-cicd-example-fig5-0.png){ width="700" } </figure>

This file defines a complete GitLab CI/CD pipeline for automating Terraform operations. It includes the following stages:

- Validate – Checks Terraform code formatting.

- Plan – Generates and summarizes the Terraform execution plan.

- Deploy – Applies changes manually to the infrastructure on the main branch.

- Cleanup – Destroys infrastructure manually if needed.

The pipeline uses a predefined Docker image (`danischm/nac:0.1.5`) and securely stores Terraform state files using GitLab's HTTP backend with locking support.

Here is the full content of the file for reference:

```yaml
---
image:
  name: danischm/nac:0.1.5
  pull_policy: if-not-present

stages:
  - validate
  - plan
  - deploy
  - cleanup

variables:
  TF_HTTP_USERNAME:
    description: "GitLab Username"
    value: "root"
  TF_HTTP_PASSWORD:
    description: "GitLab Access Token"
    value: "${GITLAB_TOKEN}"
  TF_HTTP_ADDRESS:
    description: "GitLab HTTP Address to store the TF state file"
    value: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/nac-ise-comprehensive-example"
  TF_HTTP_LOCK_ADDRESS:
    description: "GitLab HTTP Address to lock the TF state file"
    value: ${TF_HTTP_ADDRESS}/lock
  TF_HTTP_LOCK_METHOD:
    description: "Method to lock TF state file"
    value: POST
  TF_HTTP_UNLOCK_ADDRESS:
    description: "GitLab HTTP Address to unlock the TF state file"
    value: ${TF_HTTP_ADDRESS}/lock
  TF_HTTP_UNLOCK_METHOD:
    description: "Method to unlock TF state file"
    value: DELETE
  GODEBUG:
    description: "Disable IPv6 for terraform operations"
    value: net.ipv6=off

# DNS patch
before_script:
  - cp /etc/resolv.conf /tmp/resolv.conf
  - sed -i 's/8\.8\.8\.8/1.1.1.1/g' /tmp/resolv.conf
  - cp /tmp/resolv.conf /etc/resolv.conf

cache:
  key: terraform_modules_and_lock
  paths:
    - .terraform
    - .terraform.lock.hcl

validate:
  stage: validate
  script:
    - set -o pipefail && terraform fmt -check |& tee fmt_output.txt
  artifacts:
    paths:
      - fmt_output.txt

plan:
  stage: plan
  script:
    - terraform init -input=false --upgrade
    - terraform plan -out=plan.tfplan -input=false
    - terraform show -no-color plan.tfplan > plan.txt
    - terraform show -json plan.tfplan | jq > plan.json
    - terraform show -json plan.tfplan | jq '([.resource_changes[]?.change.actions?]|flatten)|{"create":(map(select(.=="create"))|length),"update":(map(select(.=="update"))|length),"delete":(map(select(.=="delete"))|length)}' > plan_gitlab.json
  artifacts:
    paths:
      - plan.json
      - plan.txt
      - plan.tfplan
      - plan_gitlab.json
    reports:
      terraform: plan_gitlab.json
  dependencies: []
  needs:
    - validate
  only:
    - merge_requests
    - main

deploy:
  stage: deploy
  script:
    - terraform init -input=false
    - terraform apply -input=false -auto-approve plan.tfplan
  dependencies:
    - plan
  needs:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual

cleanup:
  stage: cleanup
  script:
    - terraform init -input=false
    - terraform destroy -input=false -auto-approve
  dependencies: []
  needs:
    - deploy
  when: manual
  only:
    - main
```

For more information about `.gitlab-ci.yml` and other examples, see: [.gitlab-ci.yml](https://docs.gitlab.com/ee/ci/yaml/)

## Step 4: Changing Terraform Backend

When using Runners it becomes even more important to use a remote backend. The reason is that each job in the pipeline is ran by a new, immutable container which runs a specific task and dies. The Terraform state would be lost forever without a remote backend. Navigate to `main.tf` and add a remote backend section for terraform. This instructs Terraform to make use of a remote backend.

```hcl
terraform {
  backend "http" {
    skip_cert_verification = true
  }
}
```

Your `main.tf` file should now look like this:

```hcl
terraform {
  required_providers {
    ise = {
      source  = "CiscoDevNet/ise"
      version = "0.2.14"
    }
  }
}

terraform {
  backend "http" {
    skip_cert_verification = true
  }
}

module "ise" {
  source  = "netascode/nac-ise/ise"
  version = "0.2.2"

  yaml_directories = ["data/"]
}
```

Save the updated `main.tf` file (Ctrl + S) and continue with the next step.

!!! note
    Note that there are multiple options for remote backends such as AWS S3, Consul or Terraform Cloud. In this lab you will use the Terraform backend provided by GitLab.

## Step 5: Managing variables

As this code will be stored centrally in a GitLab repository, it is a good practice to replace any credentials or sensitive data with variables. These variables are securely managed in GitLab and are passed to the Runner as environment variables during pipeline execution.

In this case, the required variables have already been defined at the group level in GitLab and can be found here: [Group CI/CD Variables](https://198.18.133.101/groups/sda_as_code/-/settings/ci_cd#ci-variables)

<figure markdown>
  ![](./assets/nac-ise-cicd-example-variables_ise.png){ width="800" }
</figure>

You don’t need to redefine them in your project, but you can review or override them if needed by going to your project in GitLab and navigating to `Settings` > `CI/CD` > `Variables`

Nothing needs to be changed here — you can leave the existing configuration as it is and continue with the next step.

## Step 6: Pushing the code

Prior to committing and pushing your changes to GitLab, take a moment to inspect all the `*.yaml` files in the `data` directory. These files form the ISE configuration, providing network access configuration for two endpoints (`Host01` and `Host02`), including elements such as Policy Set, Authentication Rules, Authorization Rules, Authorization Profiles, SGTs and more:

```cli
├── data
│   ├── identity_management.nac.yaml
│   ├── network_access.nac.yaml
│   └── trust_sec.nac.yaml
└── main.tf
```

**identity_management.nac.yaml**

Contains endpoint identity groups: `CampusEndpoint01` and `CampusEndpoint02` and endpoints `Host01` and `Host02`.

**network_access.nac.yaml**

Defines network access configurations:

- **Downloadable ACL:** `PERMIT_Dacl`
- **Authorization Profiles:** `Host01_access` and `Host02_access` which grant access to both hosts, assign VLANs, and apply a downloadable access control list.
- **Policy Set:** `Wired Access Policy` applies when the device location equals "All Locations" in the DEVICE inventory
- **Authentication Rules:** `DOT1x_wired` and `MAB_wired`
- **Authorization Rules:**
    - `Host01 Authorization` - Grants access to endpoints in `CampusEndpoints01` and applies the `Host01_access` profile with `SGT1` security group.
    - `Host02 Authorization` - Grants access to endpoints in `CampusEndpoints02` and applies the `Host02_access` profile with `SGT2` security group.

**trust_sec.nac.yaml**

Contains TrustSec configuration:

- Security Groups: `SGT1` and `SGT2`
- TrustSec Policy Matrix Entries to deny traffic between `SGT1` and `SGT2`

---

To explore additional resources available for Cisco ISE Configuration, please refer to the [Data Model Documentation](https://netascode.cisco.com/docs/data_models/ise/overview).

---

Before committing changes, the Git username and Email address must be provided. This is how changes can be tracked to an individual user. Open a terminal in Visual Studio Code by navigating to `Terminal` and `New Terminal` and run the following commands:

```cli
git config --global user.name "First Last"
git config --global user.email "first@example.com"
```

In the Explorer, right-click and select **Open in Integrated Terminal**  to open a new terminal from a folder.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-terminal.png){ width="450" }
</figure>

Once the terminal is open, run the following commands to commit and push your changes to Gitlab:

 ```cli
 git add .
 git commit -m "modified backend"
 git push
 ```

!!! note
    Note that this is a commit against the main branch. It is advised to work with branching when using this in production.

The GitLab repository now contains your code and will trigger the pipeline as described in `.gitlab-ci.yml`.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-repo.png){ width="800" }
</figure>

## Step 7: Deploying the configuration

This pipeline assumes a Continuous Delivery step whereby the code is checked automatically, but requires human intervention to manually trigger the deployment of the changes. Open the project in GitLab and navigate to `Build` > `Pipelines` [link](https://198.18.133.101/sda_as_code/nac-ise-comprehensive-example/-/pipelines). The pipeline triggered by the `push` in step 5 should be visible here.

If previous steps were executed correctly, the pipeline will have completed three steps successfully and will be in a `blocked` state, meaning human intervention is required to proceed.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-initial-pipeline.png){ width="800" }
</figure>

Before clicking `deploy` it is recommended to navigate to the pipeline and verify the output of the individual jobs by clicking on each completed stage. The output of the `plan` stage will provide an overview of the planned actions.

If any of these steps have failed, the detailed stage view should also provide you the output of any jobs within that stage, providing details to help further troubleshoot.

The plan job generated a plan to add 23 resources as shown by the output:

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-build-job.png){ width="800" }
</figure>

When satisfied with the output of the plan job, you can manually trigger the deploy job by clicking play icon and then deploy. This will trigger a `terraform apply` action and push the configuration to ISE.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-deploy.png){ width="800" }
</figure>

Navigate to Cisco ISE GUI [link](https://198.18.133.27) to verify that the configuration was deployed successfully.

Go to `Policy` > `Policy Sets` > `Wired Access Policy`
<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig6.png){ width="800" }
</figure>
<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig11.png){ width="800" }
</figure>

Go to `Work Centers` > `TrustSec` > `TrustSec Policy` > `Egress Policy` > `Matrix`

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig12.png){ width="800" }
</figure>

## Step 8: Adding configuration

Now that the initial configuration is pushed to the repository and the terraform state is available centrally in GitLab, you or someone else in your team can commit new configurations, which would trigger a new pipeline when pushed to the repository.

!!! note
    Note that in production you would typically create a branch that contains any changes, which would be merged after a pull request. In this example however, you will add a new TrustSec Security Group, and commit straight to the `main` branch.

Open `data/trust_sec.nac.yaml` in Visual Studio Code and add the following section:

```yaml
      - name: SGT3
        description: SGT3
        value: 1003
```

The `trust_sec.nac.yaml` file should now look like this:

```yaml
---
ise:
  trust_sec:
    security_groups:
      - name: SGT1
        description: SGT1
        value: 1001
      - name: SGT2
        description: SGT2
        value: 1002
      - name: SGT3
        description: SGT3
        value: 1003
    matrix_entries:
      - source_sgt: SGT1
        destination_sgt: SGT2
        sgacl_name: Deny IP
      - source_sgt: SGT2
        destination_sgt: SGT1
        sgacl_name: Deny IP
```

Save the file (Ctrl + S), commit and push your changes. Either via `Visual Studio Code` or locally via command-line interface:

```cli
git add .
git commit -m "adding new sg"
git push
```

This triggers a new iteration of the pipeline that will be in `blocked` state, waiting for human intervention [link](https://198.18.133.101/sda_as_code/nac-ise-comprehensive-example/-/pipelines)

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-add-sg.png){ width="800" }
</figure>

Open the pipeline and expand the `plan` job. Note that Terraform has calculated that 1 new resource will be added.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-add-sg-build.png){ width="800" }
</figure>

Once satisfied with the proposed changes, deployment can be triggered by clicking `play` button and then `deploy`.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-deploy2.png){ width="800" }
</figure>

Navigate to Cisco ISE GUI [link](https://198.18.133.27) to verify that the new Security Group has been added:

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig7.png){ width="600" }
</figure>

## Step 9: Restoring a previous commit

Imagine that the Security Group added in step 8 resulted in an outage and you must restore the configuration to an earlier time. Because this change is tracked individually it is easy to revert to an earlier commit.

Get a list of previous commits with `git log` in a terminal. Note that Visual Studio Code has a built-in terminal which can be added to the workspace by navigating to `Terminal` and `New Terminal`.

```sh
git log --pretty=format:"%h%x09%an%x09%ad%x09%s"
```

```
$ git log --pretty=format:"%h%x09%an%x09%ad%x09%s"
0b0778ed Admin   Thu Jan 16 01:18:27 2025 -0800  adding new sg
a4e6ce69 Admin   Wed Jan 15 14:55:11 2025 -0800  add .gitlab-ci.yml file
~output omitted~
```

Note that the output of the `git log` command also shows any previous commits made against the cloned repository, followed by the two commits made in this example.

By reverting the last commit it is possible to undo any changes associated with that commit.

```
git revert <your last commit id such as 0b0778ed>
```

A new text file will open with the default commit message. Save the file and continue.

Finally, push the changes to the upstream repository (GitLab) using the following command:

```cli
git push
```

This is just one way to revert your change. Although this is the preferred way as now you have the revert action added on top of your commit history. This allows you to track changes or even undo the revert action itself.

Navigate to your project in Gitlab and open the most recent pipeline [link](https://198.18.133.101/sda_as_code/nac-ise-comprehensive-example/-/pipelines):

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-revert-add-sg.png){ width="800" }
</figure>

Expand the plan job and verify that this Terraform plan will destroy one resource:

<figure markdown>
  ![](./assets/nac-ise-cicd-example-pipeline-revert-add-sg-build.png){ width="800" }
</figure>

Once satisfied with the proposed changes, deployment can be triggered by clicking `play` button and then `deploy`.

Navigate to Cisco ISE [Web GUI](https://198.18.133.27/) to verify that the previously added Security Group has been removed.

<figure markdown>
  ![](./assets/nac-ise-cicd-example-fig9.png){ width="800" }
</figure>

## Step 10: Create CatalystCenter repository on GitLab

Log in to GitLab instance and create new blank project [link](https://198.18.133.101/projects/new). Give the project a `Project name` for example:
```
Nac CatalystCenter Comprehensive Example
```

!!! info
    Make sure you keep the repository private and uncheck `Initialize repository with a README`

<figure markdown>
  ![](./assets/nac-cc-cicd-example-fig1.png){ width="800" }
</figure>

Click on `Create project` once you are satisfied with your configuration.

!!! note
    If you chose a different name for your repository, go to your newly created GitLab repository, click the `Code` button, then copy the URL displayed under `Clone with HTTPS` by clicking the `Copy URL` icon. You will need to use that URL in the following steps.

<figure markdown>
  ![](./assets/nac-cc-cicd-example-fig_4_1.png){ width="800" }
</figure>

Next we are going to push github repository from [Lab 2 - Fabric Deployment](./lab2_fabric_deployment.md) to our empty GitLab project. 

In the `provider "catalystcenter"` block in `main.tf` file, remove `username`, `password`, `url` and `max_timeout`, as those will be passed later as environment variables.

```hcl
provider "catalystcenter" {
}
```

Save file (Ctrl + S).

Your `main.tf` file should look like this:

```hcl
terraform {
  required_providers {
    catalystcenter = {
      source  = "CiscoDevNet/catalystcenter"
      version = "0.4.6"
    }
  }
}

provider "catalystcenter" {
}

module "catalyst_center" {
  source  = "netascode/nac-catalystcenter/catalystcenter"
  version = "0.3.0"

  yaml_directories      = ["data/"]
  templates_directories = ["data/templates/"]

  use_bulk_api = true

  write_default_values_file = "defaults.yaml"
}
```

Open folder `nac-catalystcenter-comprehensive-example` in Visual Studio Code. 

In the Explorer, right-click and select **Open in Integrated Terminal**  to open a new terminal from a folder.

Execute following commands:

- Commit changes:

```cli
git add .
git commit -m "fabric provisioned"
```

- Add Gitlab Remote:

```cli
git remote add gitlab https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example.git
```

!!! note
    If you used a different name for your repository, make sure to replace the default URL with the one you copied after creating the repository.
    
    If you forgot to save it, you can find the GitLab URL in your GitLab project:
    
      - Go to your GitLab project.
      - Click the Clone button and copy the HTTPS URL.

<figure markdown>
  ![](./assets/nac-cc-cicd-example-fig_4_1.png){ width="800" }
</figure>

Verify the Remote URLs Check that both the original GitHub remote and the new GitLab remote are correctly configured:

```cli
git remote -v
```

```cli
gitlab  https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example.git (fetch)
gitlab  https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example.git (push)
origin  https://github.com/netascode/nac-catalystcenter-comprehensive-example.git (fetch)        
origin  https://github.com/netascode/nac-catalystcenter-comprehensive-example.git (push)
```

## Step 11: Migrate Terraform State

Before pushing local repository to the GitLab remote, let's migrate terraform state to remote backend.

First navigate to `main.tf` and add a remote backend section for terraform.

```hcl
terraform {
  backend "http" {
    skip_cert_verification = true
  }
}
```

Save file (Ctrl + S).

Your `main.tf` file should look like this:

```hcl
terraform {
  required_providers {
    catalystcenter = {
      source  = "CiscoDevNet/catalystcenter"
      version = "0.4.6"
    }
  }
}

terraform {
  backend "http" {
    skip_cert_verification = true
  }
}

provider "catalystcenter" {
}

module "catalyst_center" {
  source  = "netascode/nac-catalystcenter/catalystcenter"
  version = "0.3.0"

  yaml_directories      = ["data/"]
  templates_directories = ["data/templates/"]

  use_bulk_api = true

  write_default_values_file = "defaults.yaml"
}
```

Next, find the project ID by opening your project in GitLab [link](https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example#), and then clicking the three dots in the top-right corner:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-fig3.png){ width="800" }
</figure>

Save that value.

Replace **${PROJECT_ID}** with your saved values in following command:

```cli
terraform init -migrate-state `
-backend-config="address=https://198.18.133.101/api/v4/projects/${PROJECT_ID}/terraform/state/nac-catalystcenter-comprehensive-example" `
-backend-config="lock_address=https://198.18.133.101/api/v4/projects/${PROJECT_ID}/terraform/state/nac-catalystcenter-comprehensive-example/lock" `
-backend-config="unlock_address=https://198.18.133.101/api/v4/projects/${PROJECT_ID}/terraform/state/nac-catalystcenter-comprehensive-example/lock" `
-backend-config="username=root" -backend-config="password=glpat-zGWadcrQ52yo8HVMP3eN" `
-backend-config=lock_method=POST -backend-config="unlock_method=DELETE" -backend-config="retry_wait_min=5"
```

Run the migration command that will move your Terraform state from local to GitLab HTTP remote backend:

```cli
terraform init -migrate-state `
-backend-config="address=https://198.18.133.101/api/v4/projects/19/terraform/state/nac-catalystcenter-comprehensive-example" `
-backend-config="lock_address=https://198.18.133.101/api/v4/projects/19/terraform/state/nac-catalystcenter-comprehensive-example/lock" `
-backend-config="unlock_address=https://198.18.133.101/api/v4/projects/19/terraform/state/nac-catalystcenter-comprehensive-example/lock" `
-backend-config="username=root" -backend-config="password=glpat-zGWadcrQ52yo8HVMP3eN" `
-backend-config=lock_method=POST -backend-config="unlock_method=DELETE" -backend-config="retry_wait_min=5"
```

!!! note
    Make sure you use your Project ID

You will be asked do you want to copy existing state to the new backend. Type `yes` to continue. Upon success you should receive the following output:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-backend.png){ width="700" }
</figure>

Open your project in GitLab [link](https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example#), and then go to the **Operate** section in the left sidebar menu and select **Terraform states** to check if the Terraform state was migrated properly. You should see one new Terraform state:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-backend3.png){ width="800" }
</figure>

## Step 12: Adding CatalystCenter variables

In step 10, you removed `username`, `password`, `url` and `max_timeout` from `provider "catalystcenter"` block in `main.tf` file. As this code will be stored centrally in a GitLab repository, it's a best practice to replace any credentials or sensitive data with variables. These variables are securely managed in GitLab and passed to the Runner as environment variables during pipeline execution.

In this case, the required variables have already been defined at the group level in GitLab, so you don't need to create them yourself. You can find them here: [Group CI/CD Variables](https://198.18.133.101/groups/sda_as_code/-/settings/ci_cd#ci-variables)

<figure markdown>
  ![](./assets/nac-ise-cicd-example-variables_cc.png){ width="800" }
</figure>

You don’t need to redefine them in your project, but you can review or override them if needed by going to your project in GitLab and navigating to `Settings` > `CI/CD` > `Variables`

Nothing needs to be changed here — you can leave the existing configuration as it is and continue with the next step.

## Step 13: Push Code to Gitlab Remote

Commit latest changes:

```cli
git add .
git commit -m "add remote backend"
```

To push your local repository to the GitLab remote, use the following command:

```cli
git push gitlab
```

You can update your Git configuration to remove the `origin` remote and replace it with `gitlab`, making gitlab the default remote for both fetch and push. Run the following commands:

```cli
git remote remove origin
git remote rename gitlab origin
git push --set-upstream origin main
```

Once the upstream is set, you can simply run `git push` for future pushes.

## Step 14: Creating a pipeline for CatalystCenter

Create a new file `.gitlab-ci.yml` in the root of this repo 

<figure markdown>
  ![](./assets/nac-cc-cicd-example-fig6-1.png){ width="800" }
</figure>

and add the following code:

```yaml
---
image:
  name: danischm/nac:0.1.5
  pull_policy: if-not-present

stages:
  - validate
  - plan
  - deploy
  - cleanup

variables:
  TF_HTTP_USERNAME:
    description: "GitLab Username"
    value: "root"
  TF_HTTP_PASSWORD:
    description: "GitLab Access Token"
    value: "${GITLAB_TOKEN}"
  TF_HTTP_ADDRESS:
    description: "GitLab HTTP Address to store the TF state file"
    value: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/nac-catalystcenter-comprehensive-example"
  TF_HTTP_LOCK_ADDRESS:
    description: "GitLab HTTP Address to lock the TF state file"
    value: ${TF_HTTP_ADDRESS}/lock
  TF_HTTP_LOCK_METHOD:
    description: "Method to lock TF state file"
    value: POST
  TF_HTTP_UNLOCK_ADDRESS:
    description: "GitLab HTTP Address to unlock the TF state file"
    value: ${TF_HTTP_ADDRESS}/lock
  TF_HTTP_UNLOCK_METHOD:
    description: "Method to unlock TF state file"
    value: DELETE
  GODEBUG:
    description: "Disable IPv6 for terraform operations"
    value: net.ipv6=off

# DNS patch
before_script:
  - cp /etc/resolv.conf /tmp/resolv.conf
  - sed -i 's/8\.8\.8\.8/1.1.1.1/g' /tmp/resolv.conf
  - cp /tmp/resolv.conf /etc/resolv.conf

cache:
  key: terraform_modules_and_lock
  paths:
    - .terraform
    - .terraform.lock.hcl

validate:
  stage: validate
  script:
    - set -o pipefail && terraform fmt -check |& tee fmt_output.txt
  artifacts:
    paths:
      - fmt_output.txt

plan:
  stage: plan
  script:
    - terraform init -input=false --upgrade
    - terraform plan -out=plan.tfplan -input=false
    - terraform show -no-color plan.tfplan > plan.txt
    - terraform show -json plan.tfplan | jq > plan.json
    - terraform show -json plan.tfplan | jq '([.resource_changes[]?.change.actions?]|flatten)|{"create":(map(select(.=="create"))|length),"update":(map(select(.=="update"))|length),"delete":(map(select(.=="delete"))|length)}' > plan_gitlab.json
  artifacts:
    paths:
      - plan.json
      - plan.txt
      - plan.tfplan
      - plan_gitlab.json
    reports:
      terraform: plan_gitlab.json
  dependencies: []
  needs:
    - validate
  only:
    - merge_requests
    - main

deploy:
  stage: deploy
  script:
    - terraform init -input=false
    - terraform apply -input=false -auto-approve plan.tfplan
  dependencies:
    - plan
  needs:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual

cleanup:
  stage: cleanup
  script:
    - terraform init -input=false
    - terraform destroy -input=false -auto-approve
  dependencies: []
  needs:
    - deploy
  when: manual
  only:
    - main

```

Save the file (Ctrl + S), commit and push to Gitlab:

```cli
git add .
git commit -m "add gitlab-ci"
git push
```

Once you push this change to GitLab, a new pipeline will be triggered for the `Nac CatalystCenter Comprehensive Example` project. Since there are no changes to the data model, the plan stage will show no changes [link](https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example/-/pipelines).

## Step 15: Change Authentication Template on Edge Switches

To leverage the ISE configuration applied by CICD pipeline we must modify the port assignment by changing authentication template on interfaces connected to `Host01` and `Host02` from `No Authentication` to `Closed Authentication`.

!!! info
    Make sure you finish ([Lab 2 - Fabric Deployment](./lab2_fabric_deployment.md)) and [step 10 - step 14](#step-10-create-catalystcenter-repository-on-gitlab) to complete this step and the subsequent ones.

Open folder `nac-catalystcenter-comprehensive-example` in Visual Studio Code and change `authentication_template_name` to `Closed Authentication` on port-assignment section under both edge devices in `devices.nac.yaml`:

**EDGE01.cisco.eu**
    ```yaml
      - name: EDGE01.cisco.eu
        hostname: EDGE01.cisco.eu
        device_ip: 198.18.130.1
        pid: C9KV-UADP-8P
        state: PROVISION
        device_role: ACCESS
        site: Global/Poland/Krakow/Bld A
        fabric_site: Global/Poland/Krakow
        fabric_roles:
          - EDGE_NODE
        port_assignments:
          - interface_name: GigabitEthernet1/0/2
            connected_device_type: "USER_DEVICE"
            data_vlan_name: "192_168_100_0-Campus"
            authenticate_template_name: "Closed Authentication"
    ```

**EDGE02.cisco.eu**
    ```yaml
      - name: EDGE02.cisco.eu
        hostname: EDGE02.cisco.eu
        device_ip: 198.18.130.2
        pid: C9KV-UADP-8P
        state: PROVISION
        device_role: ACCESS
        site: Global/Poland/Krakow/Bld A
        fabric_site: Global/Poland/Krakow
        fabric_roles:
          - EDGE_NODE
        port_assignments:
          - interface_name: GigabitEthernet1/0/3
            connected_device_type: "USER_DEVICE"
            data_vlan_name: "192_168_100_0-Campus"
            authenticate_template_name: "Closed Authentication"
    ```

Save the file (Ctrl + S), commit and push to Gitlab:

```cli
git add .
git commit -m "changed authentication template name to Closed Authentication"
git push
```

Navigate to your project in Gitlab [link](https://198.18.133.101/sda_as_code/nac-catalystcenter-comprehensive-example) and click on **Blocked** icon:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-pipeline-3.png){ width="800" }
</figure>

The pipeline will open. Next, click on the **Run** icon to deploy the configuration:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-run.png){ width="800" }
</figure>

Once the deploy stage passes, expand the deploy job and verify that Terraform applied changes to two resources:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-run2.png){ width="800" }
</figure>

!!! note
    When you change authentication template name on edge switches, it's essential to ensure that endpoints trigger authentication so they can be identified and authorized. If this process doesn't happen automatically, you might need to initiate it manually

### Triggering Authentication on Cisco ISE

Open web browser and login to Cisco Modeling Labs ([CML](https://198.18.130.34)) using following credentials:

| Device Name | URL | Username	| Password |
|-------------|-----------------|-----------| ----------|
| *Cisco Modeling Lab* | [https://198.18.130.34](https://198.18.130.34) | ```guest``` | ```CiscoLive``` |

Next, click on `SDA-Topo` Lab and open Console on both Hosts: `Host01` and `Host02`.

To open Console, right-click on `Host01` icon and select `Console`

<figure markdown>
  ![](./assets/nac-cc-comprehensive-example-console.png){ width="500" }
</figure>

Next click `OPEN CONSOLE`:

<figure markdown>
  ![](./assets/nac-cc-comprehensive-example-console2.png){ width="400" }
</figure>

Use following credentials to log in to both hosts:

| Device Name | IP Address |  Username	| Password  |
|-------------|-----------|-----------|-----------|
| *Host1* | 192.168.100.100 |  ```cisco``` | ```cisco``` |
| *Host2* | 192.168.100.200 | ```cisco``` | ```cisco``` |

<figure markdown>
  ![](./assets/nac-cc-comprehensive-example-console3.png){ width="550" }
</figure>

When enabling authentication on Cisco ISE, you have two options to ensure hosts trigger authentication. If one option fails, proceed with the other:

- Trigger traffic from hosts

Ensure hosts send traffic to initiate authentication. The easiest method is to ping the default gateway (192.168.100.1) from each host.

**Host1**
    ```cli
    cisco@Host1:~$ ping 192.168.100.1
    PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
    From 192.168.100.1 icmp_seq=1 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=2 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=3 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=4 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=5 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=6 Destination Host Unreachable
    ```

**Host2**
    ```cli
    cisco@Host2:~$ ping 192.168.100.1
    PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
    From 192.168.100.1 icmp_seq=1 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=2 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=3 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=4 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=5 Destination Host Unreachable
    From 192.168.100.1 icmp_seq=6 Destination Host Unreachable
    ```

If the ping does not work after 2–3 minutes, proceed to the next option.

- Bounce the interface on edge switches

If the host doesn’t trigger traffic or the previous method does not work, manually reset the interface to force re-authentication. Use the following steps on the edge switch where the host is connected:

**EDGE01**
    ```cli
    EDGE01#conf t
    EDGE01(config)#interface GigabitEthernet1/0/2
    EDGE01(config-if)#shutdown
    EDGE01(config-if)#no shutdown
    ```

**EDGE02**
    ```cli
    EDGE02#conf t
    EDGE02(config)#interface GigabitEthernet1/0/3
    EDGE02(config-if)#shutdown
    EDGE02(config-if)#no shutdown
    ```

### Check Authentication Session Details on Edge Switches

After triggering authentication, you can check the authentication session details on the switch using the following commands:

**EDGE01**
    ```cli
    EDGE01#show authentication sessions int gi1/0/2 details
                Interface:  GigabitEthernet1/0/2
                  IIF-ID:  0x1CFD5F26
              MAC Address:  5254.006d.fc09
            IPv6 Address:  Unknown
            IPv4 Address:  192.168.100.100
                User-Name:  52-54-00-6D-FC-09
                      VRF:  Campus
                  Status:  Authorized
                  Domain:  DATA
          Oper host mode:  multi-auth
        Oper control dir:  both
          Session timeout:  N/A
      Acct update timeout:  172800s (local), Remaining: 172669s
        Common Session ID:  016510AC0000000CBC0FF5ED
          Acct Session ID:  0x00000002
                  Handle:  0xf4000002
          Current Policy:  PMAP_DefaultWiredDot1xClosedAuth_1X_MAB


    Server Policies:
              Vlan Group:  Vlan: 1022
                  ACS ACL: xACSACLx-IP-PERMIT_Dacl-679ca91c
                SGT Value:  1001


    Method status list:
          Method           State
            dot1x           Stopped
              mab           Authc Success
    ```

**EDGE02**
    ```cli
    EDGE02#show authentication sessions interface gigabitEthernet 1/0/3 detail
                Interface:  GigabitEthernet1/0/3
                  IIF-ID:  0x103BB518
              MAC Address:  5254.00c4.a5c7
            IPv6 Address:  fe80::5054:ff:fec4:a5c7
            IPv4 Address:  192.168.100.200
                User-Name:  52-54-00-C4-A5-C7
                      VRF:  Campus
                  Status:  Authorized
                  Domain:  DATA
          Oper host mode:  multi-auth
        Oper control dir:  both
          Session timeout:  N/A
      Acct update timeout:  172800s (local), Remaining: 172723s
        Common Session ID:  026610AC0000000CBC107ECA
          Acct Session ID:  0x00000002
                  Handle:  0x64000002
          Current Policy:  PMAP_DefaultWiredDot1xClosedAuth_1X_MAB


    Server Policies:
              Vlan Group:  Vlan: 1022
                  ACS ACL: xACSACLx-IP-PERMIT_Dacl-679ca91c
                SGT Value:  1002


    Method status list:
          Method           State
            dot1x           Stopped
              mab           Authc Success
    ```

!!! note
    `mab` should show "Authc Success" for the session to be considered successfully authenticated.

### Check ISE Radius Live Logs

- Log in to the Cisco ISE [Web GUI](https://198.18.133.27/)

- In the menu, go to: **Operations > RADIUS > Live Logs**

- Enter the MAC address of `Host01` - **52:54:00:6D:FC:09** in the filter field on **Identity** column and press Enter.

<figure markdown>
  ![](./assets/nac-cc-cicd-example-host1.png){ width="800" }
</figure>

- Click on Details:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-host1-1.png){ width="500" }
</figure>

- Review Authentication Results and confirm success

- Enter the MAC address of `Host02` - **52:54:00:C4:A5:C7** in the filter field on **Identity** column and press Enter.

<figure markdown>
  ![](./assets/nac-cc-cicd-example-host2.png){ width="800" }
</figure>

- Click on Details:

<figure markdown>
  ![](./assets/nac-cc-cicd-example-host2-1.png){ width="500" }
</figure>

## Step 16: Verify Traffic Blocking and TrustSec Configuration

### Verify Traffic Blocking Between Hosts Using Ping

Open web browser and login to Cisco Modeling Labs ([CML](https://198.18.130.34))

Next open Console on both Hosts: `Host01` and `Host02`, and check traffic blocking between them with `ping` command.

Use following credentials to log in to both hosts:

| Device Name | IP Address |  Username	| Password  |
|-------------|-----------|-----------|-----------|
| *Host1* | 192.168.100.100 |  ```cisco``` | ```cisco``` |
| *Host2* | 192.168.100.200 | ```cisco``` | ```cisco``` |

Execute the following commands:

**Host1**

```cli
cisco@Host1:~$ ping 192.168.100.200
PING 192.168.100.200 (192.168.100.200) 56(84) bytes of data.
^C
--- 192.168.100.200 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 1008ms
```

**Host2**

```cli
cisco@Host2:~$ ping 192.168.100.100
PING 192.168.100.100 (192.168.100.100) 56(84) bytes of data.
^C
--- 192.168.100.100 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 1008ms
```

### Verify TrustSec Configuration

Since the ping is blocked, the next step is to verify the TrustSec configuration on the edge switches. To do this, you need to log in to the edge switches and display role-based access control list permissions and statistics.

Use an SSH client such as `PuTTY` to connect to edge switches using the following credentials:

| Device Name | IP Address |  Username	| Password  | Enable Password  |
|-------------|-----------|-----------|-----------|-----------|
| *EDGE01* | 198.18.130.1 |  ```dnacadmin``` | ```C1sco12345``` | ```C1sco12345``` |
| *EDGE02* | 198.18.130.2 |  ```dnacadmin``` | ```C1sco12345``` | ```C1sco12345``` |

Run `show cts role-based permissions` and `show cts role-based counters` on `EDGE01` and `EDGEO2`

**EDGE01**

```cli
EDGE01#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
IPv4 Role-based permissions from group 1002:SGT2 to group 1001:SGT1:
        Deny IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE

EDGE01#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          11230      113708     0          0         
1002    1001    0          9          0          0          0          0   
```

**EDGE02**

```cli
EDGE02#show cts role-based permissions
IPv4 Role-based permissions default:
        Permit IP-00
IPv4 Role-based permissions from group 1001:SGT1 to group 1002:SGT2:
        Deny IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE

EDGE02#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          10960      113322     0          0         
1001    1002    0          68         0          0          0          0         
```
