# Problem Statement

In today's hyper-connected enterprise, the business landscape is no longer solely dominated by human interactions. We live in a world where **Non-Human Identities (NHIs)** such as automated scripts, service accounts, IoT devices, and increasingly, Agentic AI systems, are autonomously accessing business-critical applications and performing essential functions.

Many non-human identities (NHIs) and AI agents depend on static credentials and long-lived tokens for authentication. These unmanaged, persistent secrets act like a "master key" that can access business-critical applications anytime it wants, creating a major security weakness.

Beyond direct financial impact, organizations face immense reputational damage, customer churn, and severe regulatory penalties. The **Internet Archive**, for instance, experienced a series of breaches tied to unrotated static tokens in October 2024, highlighting the real-world implications of poor NHI governance.

Organizations must urgently move beyond antiquated credential management practices for their automated and AI workforces.

-----

# Introduction

This article shows a real-world use-case implementation of securing **GitHub Actions (A Non-Human Identity - NHI)**. The implementation documented here enhances security and allows short-lived access. Britive authorization profiles provide business context by tying GitHub Actions to an identity. It allows all automated actions performed by GitHub Actions to be tracked and logged.

-----

# Pre-Requisites

The implementation starts from integrating GitHub with Britive using **OIDC**. Engineers and architects should also have a good understanding of the following Britive concepts:

  * **Britive Profile creation and authorization checkout process**
  * **Access to Britive as tenant admin**
  * **Britive Identity provider**
  * **Britive Service Identity Federation**
  * **Britive PyBritive CLI and its interworking**
  * **GitHub attribute map and audience**
  * **GitHub OIDC Subject and its mapping to the GitHub repository (repo) and branch**

Now follow the configuration steps to test this real-world scenario.

-----

# Configure GitHub as Identity Provider (IdP) in Britive UI

In Britive UI, add GitHub as IdP:

**`System Administration` → `Identity Management` → `Identity Providers`**

## GitHub IdP Setting Explained

  * **Name**: `Github`
    This is just a label for the integration. You can name it anything, but "Github" makes it clear what it's for.
  * **Description**: `Github`
    A short description, again optional, but useful for documentation.
  * **Type**: `OIDC`
    Specifies that this integration uses **OpenID Connect** for authentication.
  * **Issuer URL**: `https://token.actions.githubusercontent.com`
    This is the OIDC token issuer used by GitHub Actions. It informs Britive to validate the identity tokens coming from GitHub.
  * **Validation Window (In Seconds)**: `30`
    This sets the time window in which the short-lived token is considered valid. A short window like 30 seconds helps reduce the risk of token replay attacks.
  * **Attributes Map**:
    `Github OIDC Subject -> sub`
    This maps the GitHub OIDC token's `sub` (subject) claim to a Britive attribute called "Github OIDC Subject". This is how Britive identifies the GitHub entity (e.g., repo or workflow) making the request.
  * **Allowed Audiences**: `britive`
    This ensures that the token presented to Britive has an `aud` (audience) claim that matches "britive". GitHub Actions must be configured to request tokens with this audience.

-----

# Configure GitHub as Service Identity (NHI) in Britive UI

In Britive UI, configure the following in the GitHub OIDC federated subject attribute:

## Github OIDC Subject Explained

Enter this subject `repo:netJoints/britive-github-nhi:ref:refs/heads/main` in the Github OIDC Subject field as an example. Replace it with your GitHub repo URL.

In my example, the GitHub repo FQDN URL is as follows: `https://github.com/netJoints/britive-github-nhi`

This repo and the code saved in this repo will be triggering the GitHub Actions (NHI) checkout of Britive profile.

This attribute (`repo:netJoints/britive-github-nhi:ref:refs/heads/main`) indicates that I am informing Britive to **trust OIDC tokens** that come from this specific GitHub repository and branch (main branch in our example). Here's what each part means:

### Breakdown of the OIDC Subject Value

  * `repo:` — This prefix indicates that the token is coming from a GitHub repository.
  * `netJoints/britive-github-nhi` — This is the full name of the GitHub repository:
      * `netJoints` is the GitHub organization or user.
      * `britive-github-nhi` is the repository name.
  * `ref:refs/heads/main` — This specifies the branch:
      * `refs/heads/main` refers to the `main` branch of the repository.

By setting this value in Britive, I am saying:

"Only GitHub Actions workflows that run from the `main` branch of the `netJoints/britive-github-nhi` repository are allowed to authenticate using OIDC and assume roles or access resources in Britive."

This is a **security control** to ensure that only trusted workflows (from a specific repo and branch) can use OIDC to interact with Britive.

### Best Practices and Requirements

  * If you want to allow multiple branches or repositories, you can add more federated attributes.
  * Make sure your GitHub Actions workflow is running on the correct branch (main) and repository.
  * Ensure the **audience** in the token request matches what Britive expects (e.g., `britive`).

-----

# Attach the Service Identity to Britive Profile’s Policy

-----

# Summary of GitHub OIDC Related Configuration in Britive UI

Following is the summary of what we have configured so far:

  * **GitHub Identity Provider**: Created in the above steps with the following information:
      * GitHub OIDC subject
      * Allowed audience
  * **Britive Service Identity (NHI)**: Created in the above steps with the following information:
      * GitHub OIDC subject attributes
      * Federated Attribute: Set to `repo:netJoints/britive-github-nhi:ref:refs/heads/main`.
  * **Britive Profile’s Policy**: A profile’s policy that grants temporary access to resources.

-----

# GitHub Configuration Steps

The sample GitHub actions code is here:

`https://github.com/netJoints/britive-github-nhi`

## Sample GitHub Actions Workflow code

```yaml
name: List S3 Buckets via Britive
on:
  workflow_dispatch:
permissions:
  contents: read
  id-token: write
jobs:
  list-s3:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - name: Install pybritive
        run: pip3 install pybritive
      - name: Configure and Checkout Britive Profile
        run: |
          mkdir -p ~/.aws
          pybritive configure tenant --tenant demo --no-prompt
          pybritive configure global --tenant demo --format json --no-prompt
          pybritive configure update profile-aliases default "AWS/123750444551 (Sigma Stage)/AWS Creative Profile - S3 Full Access"
          cat << EOF > ~/.aws/credentials
          [default]
          credential_process=pybritive checkout default -m awscredentialprocess -P github-britive
          EOF
          aws sts get-caller-identity
      - name: List S3 Buckets
        run: aws s3 ls
      - name: Check In Britive Profile
        run: pybritive checkin default -P github-britive
```

In the above config, the tenant “demo” is the Britive demo tenant. Its FQDN is `https://demo.britive-app.com/`

Now run the GitHub Actions workflow manually (for testing it is manual, otherwise, it will be triggered automatically based on CI/CD or other conditions).

1.  Click on `Actions` menu item in the repo.

2.  Click `Run workflow`.

-----

# Outcome

Following output shows the GitHub workflow execution steps.

Following output shows the List S3 bucket steps within the overall workflow showcasing the NHI accessing the S3 buckets with JIT and ZSP principles.

-----

# Audit and Reporting

The following output showcases a Britive audit log entry that captures a "Profile Check Out Request" initiated by the service identity `shahzad-github-nhi`. The log provides comprehensive details, including the actor's identity, the target AWS application, and the AWS environment involved.

It also highlights the **programmatic access type**, indicating automation via GitHub Actions. This level of detail demonstrates how **Britive audit logs provide full visibility into Non-Human Identities (NHI)** and the actions they perform, ensuring robust security and traceability in automated workflows.

-----

# Conclusion

In today’s world, **Non-Human Identities (NHI)** and **Agentic AI** are increasingly accessing critical business systems to perform automated tasks. However, relying on static credentials and tokens exposes organizations to breaches from stolen keys.

**Britive** eliminates this risk by replacing permanent credentials with **just-in-time (JIT) access** and **zero standing privilege (ZSP)**. This means NHIs and AI agents get only the access they need, only when they need it, without storing any credentials in systems like GitHub.

With Britive, enterprises can quickly implement secure, temporary access for automated workflows, significantly reducing the attack surface and enhancing overall security.
