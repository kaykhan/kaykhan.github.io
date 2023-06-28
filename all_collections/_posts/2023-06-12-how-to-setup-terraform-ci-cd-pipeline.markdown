---
title:  "How to setup a Terraform CI & CD pipeline using Github Actions"
layout: post
date:   2023-06-12 13:00:00 +0100
categories: terraform github-actions ci cd pipeline infrastructure automation
---

As our terraform code grows we found it becomes harder to manage our infrastructure and feel confident about the changes we make. 

Making infrastructure changes on a local machine is risky and difficult for other team members to get involved.

We will be building a ci & cd pipeline to automate infrastructure changes using Terraform and Github actions. This will give us better visbility over our infrastructure changes.

## Goal

The terraform pipeline will cover the following:

- Review process using pull-request approval
- Checks (Format, Validate, Lint, Security)
- Generating a terraform plan and uploading it as an artifact
- Comparing expected plan with actual plan
- Github comment with planned changes
- Github comment with cost of planned changes using Infracost

## Solution 

### 1. Continious Integration Pipeline

![architecture](/images/terraform-pipeline/ci.png)

<b>[terraform-ci.yaml](https://gist.github.com/kaykhan/a9d90da7493813dbde18da70b4ad4b3a#file-terraform-ci-yaml)</b>

- Initial Setup

We create a `github/workflows/terraform-ci.yaml` file, this workflow will run only when a pull_request is made and we limit the concurrency so that only a single pipeline can be running at once.

{% highlight yaml %}
name: terraform-ci

on:
  pull_request:

concurrency:
  group: terraform
{% endhighlight %}

- Define the jobs

![github-actions-ci](/images/terraform-pipeline/github-actions-ci.png)

<b>1. Format</b>

We use the official [terraform fmt](https://developer.hashicorp.com/terraform/cli/commands/fmt) command to enforce a coding style.

{% highlight yaml %}
  format:
    name: format
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Terraform Format
      run: terraform fmt -check -recursive
{% endhighlight %}

<b>2. Validate</b>

We use the official [terraform validate](https://developer.hashicorp.com/terraform/cli/commands/validate) command to verify the syntax.

{% highlight yaml %}
{% raw %}
  validate:
    name: validate 
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: us-east-2
        aws-access-key-id: ${{ secrets.TERRAFORM_AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.TERRAFORM_AWS_SECRET_ACCESS_KEY }}
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.3.7
    - name: Terraform Init
      id: init
      run: terraform init -backend-config=prod.s3.tfbackend
    - name: Terraform Workspace
      run: terraform workspace select prod || terraform workspace new prod 
    - name: Terraform Validate
      run: terraform validate
{% endraw %}
{% endhighlight %}

<b>3. Lint</b>

We use a custom plugin [tflint](https://github.com/terraform-linters/tflint) to enforce best practises. tflint requires you to setup a `.tflint.hcl` file which you can find in their documentation.

{% highlight yaml %}
{% raw %}
  lint:
    name: lint 
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - uses: terraform-linters/setup-tflint@v3
      name: Setup
      with:
        tflint_version: v0.46.1
    - name: Init
      run: tflint --init
      env:
        GITHUB_TOKEN: ${{ github.token }}
    - name: Lint
      run: tflint --minimum-failure-severity=error
{% endraw %}
{% endhighlight %}

<b>4. Security</b>

We use a custom plugin [tfsec](https://github.com/aquasecurity/tfsec) to enforce security best practises.

{% highlight yaml %}
  security:
    name: security 
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Install
      run: curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
    - name: Check
      run: tfsec --soft-fail
{% endhighlight %}


<b>5.  Plan</b>

The Plan job is executed once the previous jobs have successfully completed.

- Generates a terraform plan
- Uploads the terraform plan as an artifact to github actions
- Adds a comment of the plan to the pull request

{% highlight yaml %}
{% raw %}
 plan:
    needs: [conflict, format, validate, lint, security]
    name: plan
    runs-on: ubuntu-latest
    outputs:
      tfplan_identifier: ${{ steps.plan.outputs.tfplan_identifier }}
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: us-east-2
        aws-access-key-id: ${{ secrets.TERRAFORM_AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.TERRAFORM_AWS_SECRET_ACCESS_KEY }}
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.3.7
    - name: Terraform Init
      id: init
      run: terraform init -backend-config=prod.s3.tfbackend
    - name: Terraform Workspace
      run: terraform workspace select prod || terraform workspace new prod 
    - name: Terraform Plan
      id: plan
      continue-on-error: true
      run: |
        tfplan_identifier=tfplan-expected
        echo "tfplan_identifier=$tfplan_identifier" >> $GITHUB_OUTPUT
        terraform plan -var-file="prod.tfvars" -input=false -no-color -out=${tfplan_identifier}
        ls
    - name: Upload Plan
      uses: actions/upload-artifact@v3
      with:
        name: ${{ steps.plan.outputs.tfplan_identifier}}
        path: ${{ steps.plan.outputs.tfplan_identifier}}

    - name: Terraform Comment
      uses: actions/github-script@v6
      env:
        PLAN_OUTPUT: "${{ steps.plan.outputs.stdout }}"
        PLAN_ERROR: "${{ steps.plan.outputs.stderr }}"
      with:
        script: |
          const body = `
            \`\`\`hcl
            ${{env.PLAN_OUTPUT}}
            ${{env.PLAN_ERROR}}
            \`\`\`
          `;

          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: body
          })
    - name: Final
      run: |
        echo ${{ steps.plan.outcome }}
        [[ ${{ steps.plan.outcome }} == 'failure' ]] && exit 1 || exit 0

{% endraw %}
{% endhighlight %}


![plan](/images/terraform-pipeline/plan.png)

<b>6. Cost</b>

We use [Infracost](https://www.infracost.io/) to calculate cost breakdown of the planned resources. 
We then comment on the PR with the cost breakdown.

{% highlight yaml %}
{% raw %}
  cost:
    needs: plan
    name: cost
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout/@v3
    - name: Setup Infracost
      uses: infracost/actions/setup@v2
      with:
        api-key: ${{ secrets.INFRACOST_API_KEY }}
    - name: Checkout base branch
      uses: actions/checkout@v3
      with:
        ref: '${{ github.event.pull_request.base.ref }}'

    - name: Generate Infracost cost estimate baseline
      run: |
        infracost breakdown --path=. \
                            --format=json \
                            --out-file=/tmp/infracost-base.json
    - name: Checkout PR branch
      uses: actions/checkout@v3
    - name: Generate Infracost diff
      run: |
        infracost diff --path=. \
                        --format=json \
                        --compare-to=/tmp/infracost-base.json \
                        --out-file=/tmp/infracost.json
    - name: Post Infracost comment
      run: |
          infracost comment github --path=/tmp/infracost.json \
                                   --repo=$GITHUB_REPOSITORY \
                                   --github-token=${{github.token}} \
                                   --pull-request=${{github.event.pull_request.number}} \
                                   --behavior=new
{% endraw %}
{% endhighlight %}


![infra-cost](/images/terraform-pipeline/infra-cost.png)

---




### 2. Continuous Deployment Pipeline

![architecture](/images/terraform-pipeline/cd.png)


<b>[terraform-cd.yaml](https://gist.github.com/kaykhan/a9d90da7493813dbde18da70b4ad4b3a#file-terraform-cd-yaml)</b>

- Initial Setup

We create a `github/workflows/terraform-cd.yaml` file, this workflow will run when our pull request is merged into master/main. 

{% highlight yaml %}
name: terraform-cd

on:
  push:
    branches: [ "master", main" ]

concurrency:
  group: terraform
{% endhighlight %}

- Define the jobs

![github-actions-cd](/images/terraform-pipeline/github-actions-cd.png)

<b>1. Compare</b>

The compare job is used to compare the `expected-plan` which has been uploaded as an artifact in terraform-ci.yaml, with the new plan generated in this job we call this new plan the `actual-plan`.
If the expected resource changes do not match, the job will fail causing the pipeline to fail. 

{% highlight yaml %}
{% raw %}
  compare:
    needs: [format, validate, lint, security]
    name: compare
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: us-east-2
        aws-access-key-id: ${{ secrets.TERRAFORM_AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.TERRAFORM_AWS_SECRET_ACCESS_KEY }}
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.3.7
        terraform_wrapper: false
    - name: Terraform Init
      id: init
      run: terraform init -backend-config=prod.s3.tfbackend
    - name: Terraform Workspace
      run: terraform workspace select prod || terraform workspace new prod 
    - name: Download Plan
      id: download-plan
      uses: dawidd6/action-download-artifact@v2
      with:
        github_token: ${{ github.token }}
        workflow: terraform-ci.yaml
        name: tfplan-expected
    - name: Terraform Compare
      id: plan
      run: |
        terraform plan -var-file="prod.tfvars" -input=false -no-color -out="tfplan-actual"
        terraform show -json -no-color tfplan-actual | jq '.resource_changes' > actual.json
        terraform show -json -no-color tfplan-expected | jq '.resource_changes' > expected.json
        echo "Diff check"
        git diff --no-index expected.json actual.json | cat
        git diff --no-index expected.json actual.json --exit-code 
{% endraw %}
{% endhighlight %}



<b>2. Apply</b>

We finally apply the changes with auto-approve

{% highlight yaml %}
{% raw %}
  apply:
    needs: [format, validate, lint, security, compare]
    name: apply
    runs-on: ubuntu-latest
    # Add a manual required-reviewer once we are on Github Enterprise
    environment: production
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: us-east-2
        aws-access-key-id: ${{ secrets.TERRAFORM_AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.TERRAFORM_AWS_SECRET_ACCESS_KEY }}
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.3.7
    - name: Terraform Init
      id: init
      run: terraform init -backend-config=prod.s3.tfbackend
    - name: Terraform Workspace
      run: terraform workspace select prod || terraform workspace new prod 
    - name: Terraform Plan
      id: plan
      run: |
        terraform plan -var-file="prod.tfvars" -input=false -no-color
    - name: Terraform Apply
      run: terraform apply -var-file="prod.tfvars" -auto-approve -input=false
{% endraw %}
{% endhighlight %}

---

### 3. Repository Setup

Create a new branch rule with the following settings:

![terraform-repo-setup](/images/terraform-pipeline/repo_setup.png)

---
## Potential Improvements

1. Use github's required-reviwer approval process to add a manual approval before the apply stage is run within terraform-cd.yaml. (Requires Github Enterprise Licence) [learn more](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#required-reviewers)
2. In `terraform-cd.yaml` pass the `actual-plan` to the `apply` job rather than regenerating a plan.

## References
