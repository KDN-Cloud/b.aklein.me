---
layout: post
toc: true
promote_deployonfriday: true
title: "Connecting Authentik to AWS Identity Center with SAML and SCIM"
date: 2026-07-06
author: Anthony Klein
description: "A real account of wiring Authentik into AWS Identity Center as an external IdP using SAML for authentication and SCIM for user and group provisioning, including the frustrating CSRF issue that blocked everything until I added the correct AWS regional endpoint."
tags:
  - authentik
  - authentik-sso
  - authentik-saml
  - authentik-scim
  - authentik-aws
  - authentik-identity-center
  - authentik-provider
  - authentik-application
  - authentik-backchannel
  - authentik-backchannel-provider
  - authentik-csrf
  - authentik-trusted-origins
  - authentik-external-idp
  - authentik-homelab
  - authentik-self-hosted
  - authentik-docker
  - authentik-configuration
  - authentik-integration
  - authentik-setup
  - authentik-saml-provider
  - authentik-scim-provider
  - authentik-user-provisioning
  - authentik-group-provisioning
  - authentik-nameID
  - authentik-acs
  - authentik-issuer
  - aws
  - aws-identity-center
  - aws-sso
  - aws-saml
  - aws-scim
  - aws-iam-identity-center
  - aws-identity-center-saml
  - aws-identity-center-scim
  - aws-identity-center-authentik
  - aws-identity-center-external-idp
  - aws-identity-center-setup
  - aws-identity-center-integration
  - aws-identity-center-configuration
  - aws-us-east-1
  - aws-regional-endpoint
  - aws-sso-portal
  - aws-console-access
  - aws-iam
  - aws-federation
  - aws-saml-integration
  - saml
  - saml2
  - saml-provider
  - saml-assertion
  - saml-acs
  - saml-issuer
  - saml-post-binding
  - saml-nameID
  - saml-email
  - saml-certificate
  - saml-signing
  - saml-federation
  - saml-sso
  - saml-idp
  - saml-sp
  - saml-troubleshooting
  - scim
  - scim-provisioning
  - scim-user-provisioning
  - scim-group-provisioning
  - scim-aws
  - scim-authentik
  - scim-backchannel
  - scim-token
  - scim-endpoint
  - scim-sync
  - scim-auto-provisioning
  - sso
  - single-sign-on
  - identity-provider
  - idp
  - external-idp
  - oidc
  - oauth2
  - csrf
  - csrf-trusted-origins
  - trusted-origins
  - regional-endpoint
  - homelab
  - homelab-aws
  - homelab-sso
  - homelab-auth
  - homelab-security
  - homelab-identity
  - homelab-infrastructure
  - self-hosted
  - self-hosted-idp
  - self-hosted-sso
  - docker
  - docker-compose
  - proxmox
  - linux
  - sysadmin
  - sre
  - infrastructure
  - devops
  - security
  - security-engineering
  - iam
  - identity-access-management
  - cloud
  - cloud-security
  - cloud-access
  - federation
  - kdn-lab
  - variablenix
---

Getting Authentik wired into AWS Identity Center took me longer than it should have. The integration itself is not complicated once you understand what goes where, but the documentation is just vague enough to send you in the wrong direction on a few key steps. And there is one specific thing, one environment variable, that is not documented anywhere prominent and will silently block your SCIM provisioning until you figure it out.

This post covers the full setup: SAML for authentication, SCIM for user and group provisioning, and the `AUTHENTIK_CSRF_TRUSTED_ORIGINS` fix that finally made everything work.

The [official Authentik integration guide for AWS](https://integrations.goauthentik.io/cloud-providers/aws/) is a reasonable starting point and covers the basic provider and application wiring. What it does not cover is the CSRF trusted origins requirement or the NameID and SCIM username alignment issue tracked in [GitHub issue #4554](https://github.com/goauthentik/authentik/issues/4554). This post fills those gaps.

This is part of an ongoing Authentik series on AK // SYS LOG. The foundation post is [Authentik SSO: Replacing Every Password Prompt in My Homelab](https://b.aklein.me/2026/07/05/authentik-sso-homelab.html).

---

## What You Are Building

The end state is Authentik acting as an external identity provider for AWS Identity Center. Users log into the AWS access portal, get redirected to Authentik for authentication, and land in the AWS console with the right permissions. SCIM keeps users and groups in sync automatically between Authentik and AWS Identity Center so you are not managing two separate user directories.

Two providers are involved on the Authentik side:

- A **SAML provider** handles the authentication flow
- A **SCIM provider** handles user and group provisioning as a backchannel provider

Both get wired together through a single Authentik application.

---

## The CSRF Problem (Read This First)

Before getting into the setup steps, I want to call out the thing that cost me the most time.

SCIM provisioning from AWS Identity Center to Authentik kept failing. Users were not syncing. The SCIM test in AWS would return errors. I went through the provider config multiple times, checked the token, verified the endpoint URL, and could not find anything wrong.

The actual problem was a missing entry in `AUTHENTIK_CSRF_TRUSTED_ORIGINS`.

The way I found it was the Authentik server logs. While everything looked configured correctly on the surface, the logs were showing 403 rejections from Django's CSRF middleware. Something like:

```
WARNING django.security.csrf Forbidden (Referer checking failed - https://us-east-1.sso.signin.aws does not match any trusted origins.): /api/v3/core/users/
```

That line makes the fix obvious once you see it. The AWS regional SSO endpoint was making requests to Authentik and Django was rejecting them because that origin was not in the trusted list. If you are running Graylog or any centralized log aggregation, this will show up there. Otherwise check the container logs directly:

```bash
docker logs authentik-server 2>&1 | grep -i csrf
```

When AWS Identity Center makes SCIM requests to Authentik, those requests originate from AWS's regional SSO infrastructure. Authentik's CSRF protection does not recognize that origin by default and rejects the requests. The fix is adding the AWS regional SSO endpoint for your Identity Center region to the trusted origins list in your Authentik environment config.

For us-east-1:

```bash
AUTHENTIK_CSRF_TRUSTED_ORIGINS="https://auth.lab.example.com,https://us-east-1.sso.signin.aws"
```

The regional URL format is always `https://<region>.sso.signin.aws`. Replace the region with wherever your AWS Identity Center instance lives. If your Identity Center is in us-west-2, it would be `https://us-west-2.sso.signin.aws`, respectively.

Without this, SCIM appears to be configured correctly but quietly fails. Once I added it and restarted the Authentik server container, SCIM started syncing immediately.

Add it to your `.env` file and restart:

```bash
docker compose restart server worker
```

---

## AWS Identity Center Setup

Start on the AWS side. You need to configure Identity Center to use an external IdP before Authentik has anything to connect to.

In the AWS Console go to **IAM Identity Center > Settings > Identity source**. Click **Actions > Change identity source** and select **External identity provider**.

AWS will give you two things you need:

- **IdP sign-in URL**: this is the AWS portal URL your users will hit. Copy it, you will use it as the Launch URL in Authentik.
- **Service provider metadata**: download this XML file or copy the individual values:
  - **ACS URL** (Assertion Consumer Service URL)
  - **Entity ID / Issuer**

Keep these handy. You will paste them into Authentik shortly.

AWS will also ask you to upload the IdP metadata from Authentik. You can come back to this step once the Authentik SAML provider is configured.

For SCIM, stay in **IAM Identity Center > Settings** and find the **Automatic provisioning** section. Enable it. AWS will generate a **SCIM endpoint URL** and an **access token**. Copy both. The token is only shown once.

---

## Authentik Setup

### SAML Provider

In Authentik go to **Admin > Applications > Providers > Create** and select **SAML Provider**.

Fill in:

- **Name**: something descriptive like `AWS SAML`
- **ACS URL**: paste the ACS URL from AWS
- **Issuer**: set this to your Authentik FQDN, `https://auth.lab.example.com`. This is the value that tells AWS who is sending the SAML assertion. Do not use the AWS entity ID here even if Authentik pre-fills it.
- **Audience**: paste the AWS Entity ID / Issuer value from the AWS metadata
- **Binding**: `Post`
- **Name ID Policy**: `Email`
- **Name ID Property Mapping**: `authentik default SAML Mapping: Email`

Under **Advanced Protocol Settings**:

- **Signing Certificate**: select your Authentik self-signed certificate
- **Sign assertions**: enabled
- **Sign responses**: enabled

{% include note.html content="The Issuer field in Authentik sometimes pre-fills with the AWS value after you paste the ACS URL. This is wrong. Your Issuer must be your own Authentik FQDN, not the AWS endpoint. AWS uses this value to identify who sent the assertion and it needs to match what you register as your IdP metadata on the AWS side." %}

Save the provider. Then go to **Admin > Applications > Providers**, find the SAML provider you just created, and click **Download metadata**. This downloads an XML file that contains your Authentik public certificate, the SSO URL, and the entity ID.

Back in AWS, go to **IAM Identity Center > Settings > Identity source** and click **Actions > Manage identity source** (or edit the external IdP you set up earlier). There will be a field to upload the IdP SAML metadata. Upload the XML file you just downloaded from Authentik. AWS parses it automatically and populates the certificate and endpoint fields from it. This completes the trust relationship so AWS knows how to validate SAML assertions coming from your Authentik instance.

### SCIM Provider

Go to **Admin > Applications > Providers > Create** and select **SCIM Provider**.

Fill in:

- **Name**: something like `AWS SCIM`
- **URL**: paste the SCIM endpoint URL from AWS Identity Center
- **Token**: paste the access token from AWS
- **Verify SSL**: enabled

Under **User filtering**, set the property mappings to control which attributes get synced.

The defaults cover most attributes but AWS Identity Center has a specific requirement worth knowing: the `userName` attribute must map to the user's email address. This is what AWS uses to match a SCIM-provisioned user to an authenticated SAML user. If these do not agree, AWS cannot reconcile the two and login fails even though both the user and the assertion exist correctly.

The default Authentik SCIM mapping for `userName` maps to the Authentik username, not the email. If your Authentik usernames are email addresses, the default works fine. If your usernames are not email addresses, you need to create a custom SCIM property mapping that resolves `userName` to `request.user.email` instead.

AWS also does not support all SCIM attributes. The `photos` attribute in particular will cause sync errors if included. If you run into SCIM sync failures after provisioning starts, check **Admin > System Tasks** in Authentik for the specific attribute causing the issue and remove it from the property mappings.

Save the provider.

### Authentik Application

Go to **Admin > Applications > Applications > Create**.

Fill in:

- **Name**: `AWS Identity Center`
- **Slug**: `aws-identity-center`
- **Provider**: select the SAML provider you created
- **Launch URL**: paste the AWS IdP sign-in URL from earlier
- **Backchannel Providers**: select the SCIM provider

The backchannel provider field is the part that is not obvious from the docs. It is on the application, not on either provider. This is how Authentik knows to run SCIM provisioning whenever users or groups are assigned to this application.

Save the application.

---

## User and Group Assignment

For SCIM to sync anything to AWS, users and groups need to be assigned to the AWS Identity Center application in Authentik.

Go to the application and open **Policy / Bindings**. Add group bindings for whatever groups should have AWS access. In my setup I have an `AWS Staff` group and an `AWS ReadOnly` group, both bound to the application.

Once a group is bound, Authentik pushes those users to AWS Identity Center via SCIM automatically. You should see them appear in **IAM Identity Center > Users** within a minute or two.

To confirm the sync worked, go to **IAM Identity Center > Users** in the AWS Console. Your Authentik users should be listed there with their email addresses and display names. Groups will appear under **IAM Identity Center > Groups**. If nothing shows up after a couple of minutes, go to **Admin > System Tasks** in Authentik and look for any failed SCIM sync tasks. The error messages there are usually specific enough to tell you exactly what went wrong.

You can also trigger a manual sync from Authentik by going to **Admin > Applications > Providers**, finding your SCIM provider, and clicking **Sync**.

---

## Testing the Login Flow

With everything configured, go to the AWS access portal URL and try signing in. The flow should be:

1. AWS access portal redirects to Authentik
2. Authentik presents the login page
3. After authenticating, Authentik sends a SAML assertion back to AWS
4. AWS validates the assertion and logs you into the portal

One thing worth knowing before you test: with an external IdP configured, AWS defers MFA entirely to Authentik. AWS does not prompt for a second factor on its own. If your Authentik authentication flow requires TOTP or WebAuthn, that prompt happens on the Authentik login page before the SAML assertion is ever sent to AWS. Once the assertion arrives at AWS, it is treated as already fully authenticated. This is the correct behaviour and means your Authentik MFA policy covers AWS access automatically.

If the login fails at step 3 or 4, the most common causes are:

**Certificate mismatch.** Make sure the signing certificate in Authentik matches what you uploaded as IdP metadata to AWS. If you have rotated certificates since uploading the metadata, re-download and re-upload the metadata XML.

**Issuer mismatch.** If the Issuer in your Authentik SAML provider does not match what AWS has registered as the IdP entity ID, the assertion will be rejected. Both sides need to agree on this value.

**Name ID format mismatch.** AWS Identity Center expects Email format for the Name ID. Confirm both the Name ID Policy and the Name ID Property Mapping in the Authentik SAML provider are set to Email.

**CSRF error on SCIM.** If authentication works but users are not provisioning, go back to the `AUTHENTIK_CSRF_TRUSTED_ORIGINS` fix at the top of this post.

**NameID and SCIM username mismatch.** This one is tracked in [GitHub issue #4554](https://github.com/goauthentik/authentik/issues/4554) and is worth knowing about. AWS Identity Center uses the NameID from the SAML assertion to match an authenticated user to a provisioned SCIM user. If your NameID policy is set to Email in the SAML provider but your SCIM provider is mapping `userName` to something other than email, AWS cannot match them and login will fail even though both the SAML assertion and the SCIM user exist. Make sure the `userName` attribute in your SCIM property mappings resolves to the same email address that the SAML NameID sends.

---

## Permissions in AWS

SCIM syncs users and groups to AWS Identity Center but it does not automatically give them access to anything. You still need to assign permission sets to groups in Identity Center and associate those with AWS accounts.

In **IAM Identity Center > AWS accounts**, select the account you want to grant access to, click **Assign users or groups**, select your synced group, and assign a permission set. The built-in `AdministratorAccess` and `ReadOnlyAccess` permission sets cover most homelab use cases without needing to create custom ones.

Once assigned, users in that group will see the account in their AWS access portal with the associated permission set available.

---

## The Thing That Would Have Saved Me Hours

The `AUTHENTIK_CSRF_TRUSTED_ORIGINS` variable is mentioned in the Authentik docs if you know to look for it, but it is not in the AWS Identity Center integration guide and it is not in the error messages you see when SCIM fails because of it. You just get silent provisioning failures.

If you are setting this up fresh, add the variable before you even test SCIM for the first time:

```bash
AUTHENTIK_CSRF_TRUSTED_ORIGINS="https://auth.lab.example.com,https://us-east-1.sso.signin.aws"
```

Adjust the region to match your Identity Center instance. That is it. Everything else in the setup is straightforward once you know where the pieces go.

