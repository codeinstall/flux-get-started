# flux-get-started

[![CircleCI](https://circleci.com/gh/fluxcd/flux-get-started.svg?style=svg)](https://circleci.com/gh/fluxcd/flux-get-started)

We published a step-by-step run-through on how to use Flux and Helm Operator [over
here](https://github.com/fluxcd/flux/blob/master/docs/tutorials/get-started-helm.md).

## Workloads

[podinfo](https://github.com/stefanprodan/podinfo)
* Kubernetes deployment, ClusterIP service and Horizontal Pod Autoscaler
* init container automated image updates (regular expression filter)
* container automated image updates (semantic versioning filter)

## Helm Releases

Mongodb
* Source: Helm repository (stable)
* Kubernetes deployment
* automated image updates (semantic versioning filter)

Redis
* Source: Helm repository (stable)
* Kubernetes stateful set 
* locked automated image updates (semantic versioning filter)

Ghost
* Source: Git repository
* disabled automated image updates (glob filter)
* has external dependency - mariadb (stable)

## Manifests Validation

CircleCI [jobs](./.circleci/config.yml):
* validate Kubernetes manifests with [kubeval](https://github.com/instrumenta/kubeval)
* validate Flux Helm Releases with [hrval](https://github.com/stefanprodan/hrval-action)

### <a name="help"></a>Getting Help

If you have any questions about, feedback for or problems with `flux-get-started`:


- Invite yourself to the <a href="https://slack.cncf.io" target="_blank">CNCF community</a>
  slack and ask a question on the [#flux](https://cloud-native.slack.com/messages/flux/)
  channel.
- To be part of the conversation about Flux's development, join the
  [flux-dev mailing list](https://lists.cncf.io/g/cncf-flux-dev).
- [File an issue.](https://github.com/fluxcd/flux/issues/new)

Your feedback is always welcome!













# Shared S3 Storage for PDF Generation

## Overview

The PDF service uses two S3 buckets. `shared-events` stores PDFs generated for RAP-app or requested by a user. These files expire after 28 days. `shared-doc-template` stores the document templates used to generate the PDFs. Templates are retained indefinitely with no automatic expiry.

This document defines the production storage design for RAP-app and the two connected applications. Values in angle brackets identify environment details that must be supplied from the deployment configuration.

| Document detail | Value |
|---|---|
| Service | PDF generation and document storage |
| Environment | Production |
| Service owner | `<SERVICE_OWNER>` |
| Platform owner | `<PLATFORM_OWNER>` |
| AWS account | `<AWS_ACCOUNT_ID>` |
| AWS region | `<AWS_REGION>` |
| Jira reference | `<JIRA_REFERENCE>` |
| Infrastructure repository | `<REPOSITORY_URL>` |

## Bucket configuration

| Setting | shared-events | shared-doc-template |
|---|---|---|
| Purpose | Generated PDF files | Source templates for PDF generation |
| Bucket ARN | `arn:aws:s3:::shared-events` | `arn:aws:s3:::shared-doc-template` |
| Region | `<AWS_REGION>` | `<AWS_REGION>` |
| Retention | 28 days | Indefinite |
| Automatic object expiry | Enabled at 28 days | Disabled |
| Storage class | S3 Standard | S3 Standard |
| Versioning design | Unversioned storage for temporary PDFs | Enabled to preserve template history |
| Encryption | SSE-KMS | SSE-KMS |
| KMS key ARN | `<EVENTS_KMS_KEY_ARN>` | `<TEMPLATE_KMS_KEY_ARN>` |
| S3 access point ARN | `<EVENTS_ACCESS_POINT_ARN>` | `<TEMPLATE_ACCESS_POINT_ARN>` |
| S3 access point alias | `<EVENTS_ACCESS_POINT_ALIAS>` | `<TEMPLATE_ACCESS_POINT_ALIAS>` |
| Public access | Blocked | Blocked |
| Object ownership | Bucket owner enforced | Bucket owner enforced |
| ACLs | Disabled | Disabled |

The bucket names above are the service identifiers. If environment naming adds a prefix or suffix then record the deployed names as `<EVENTS_BUCKET_NAME>` and `<TEMPLATE_BUCKET_NAME>` with their corresponding ARNs in the application configuration.

## PDF generation and retrieval

1. RAP-app or a connected application receives a PDF generation request and checks that the user can access the requested data.
2. The generating service reads the required template from `shared-doc-template` using its workload role.
3. The service generates the PDF and saves it to its allocated prefix in `shared-events`.
4. The application records the object key and creation time against the request. It returns the PDF through an authenticated download route or a short-lived presigned URL.
5. The application stops serving the file once its 28-day retention period has ended. S3 lifecycle processing removes the stored object.

Generated PDFs use unique object keys. Regeneration creates a new object rather than overwriting the previous file. This keeps each file's creation time and expiry clear.

End users do not receive AWS credentials. The application checks ownership before issuing a download. Presigned URLs must use a reachable endpoint and expire within `<DOWNLOAD_URL_TTL>`. They must not remain usable beyond the PDF's application expiry time.

## Application access

Each application uses a separate IAM role. PDF access is restricted to its allocated prefix. Template access is read-only and limited to the templates assigned to that application. Shared templates can be included in more than one role's allowed template scope.

| Application | Workload role ARN | shared-events scope | shared-doc-template scope |
|---|---|---|---|
| RAP-app | `<RAP_APP_ROLE_ARN>` | Read and write under `<RAP_PDF_PREFIX>` | Read under `<RAP_TEMPLATE_PREFIX>` |
| `<APPLICATION_2_NAME>` | `<APPLICATION_2_ROLE_ARN>` | `<APPLICATION_2_PDF_ACCESS_AND_PREFIX>` | `<APPLICATION_2_TEMPLATE_READ_PREFIX>` |
| `<APPLICATION_3_NAME>` | `<APPLICATION_3_ROLE_ARN>` | `<APPLICATION_3_PDF_ACCESS_AND_PREFIX>` | `<APPLICATION_3_TEMPLATE_READ_PREFIX>` |

The access scope for each connected application determines which permissions below are attached to its role. A read-only consumer receives no PDF write permissions. Access to another application's PDFs requires an explicit entry in the role policy.

### Role permissions

| Role function | S3 permissions | KMS permissions | Resource boundary |
|---|---|---|---|
| PDF generation | `s3:PutObject` and `s3:GetObject` on PDFs plus `s3:GetObject` on templates | `kms:GenerateDataKey` and `kms:Decrypt` on the events key plus `kms:Decrypt` on the template key | Assigned PDF and template prefixes |
| PDF retrieval | `s3:GetObject` | `kms:Decrypt` on the events key | Assigned PDF prefix |
| Template publishing through `<TEMPLATE_PUBLISHER_ROLE_ARN>` | `s3:PutObject` and `s3:GetObject` | `kms:GenerateDataKey` and `kms:Decrypt` on the template key | Approved template prefixes |
| Template recovery through `<TEMPLATE_RECOVERY_ROLE_ARN>` | `s3:GetObjectVersion` and `s3:PutObject` | `kms:Decrypt` and `kms:GenerateDataKey` on the template key | Template versions covered by the recovery request |
| Infrastructure deployment through `<DEPLOYMENT_ROLE_ARN>` | Bucket configuration and access point administration | Key administration under the platform key policy | The two buckets and their supporting resources |

Object listing is granted only where the application requires it. Scope `s3:ListBucket` to the appropriate access point with an `s3:prefix` condition. Services that use known object keys do not need listing access.

Where multipart uploads are used grant `s3:AbortMultipartUpload` and `s3:ListMultipartUploadParts` on the writer's assigned prefix. Application roles have no object deletion, version deletion, template publishing or bucket administration permissions.

The deployment role has no routine access to document contents. Template publishing and recovery remain separate from normal application access.

## Access points and network controls

The events access point attaches to `shared-events`. The template access point attaches to `shared-doc-template`. Their policies identify the permitted role ARNs and object prefixes. The bucket policies permit these access points and restrict application access through other paths.

Both the access point and the underlying bucket must allow the request. An access point policy alone does not prevent access through the bucket endpoint. This restriction must also be enforced at bucket level. See [AWS access point policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-policies.html).

| Network setting | Deployment value |
|---|---|
| Application runtime | `<APPLICATION_RUNTIME>` |
| Access point network origin | `<ACCESS_POINT_NETWORK_ORIGIN>` |
| Permitted VPC where applicable | `<VPC_ID>` |
| S3 endpoint where applicable | `<S3_VPC_ENDPOINT_ID>` |
| Approved network path | `<APPROVED_S3_NETWORK_PATH>` |
| Workload identity provider | `<WORKLOAD_IDENTITY_PROVIDER>` |

The network path must support the actual application runtime. A VPC-only access point requires connectivity through its configured VPC. Where users cannot reach that path the application streams the PDF through its authenticated download route.

All requests use HTTPS. Workloads use temporary role credentials through the platform identity integration. Role trust is restricted to the exact workload identity. Static AWS access keys are not stored in application configuration.

## Retention and lifecycle

### shared-events

Apply an enabled lifecycle rule to all generated PDFs with expiration set to 28 days after object creation. There are no exclusions for application prefixes. Abort incomplete multipart uploads after seven days.

The production design uses unversioned storage for this bucket so expired PDFs do not remain as noncurrent versions. If the deployed bucket already has versioning enabled or suspended then existing versions must be assessed and a version cleanup configuration agreed before the 28-day retention control is accepted. Suspending versioning does not remove historical versions.

S3 lifecycle deletion is asynchronous. Applications enforce the 28-day availability limit using the recorded creation time rather than waiting for physical deletion. The lifecycle rule is a storage cleanup control and does not guarantee deletion at an exact hour. See [AWS lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html).

Generated PDFs are not retained in a separate long-term archive. Any replica or backup containing these files must follow the same retention requirement. Files removed after expiry are no longer available for recovery from this bucket.

### shared-doc-template

Templates never expire automatically. Do not configure current-version expiry or noncurrent-version expiry for this bucket. Retain template history so an earlier version can be restored after a failed release.

Only the template publishing role can update templates. Applications use an explicit template release key so a new publication does not silently change existing generation requests. Record the template release used against each generated PDF.

Indefinite retention does not make objects immutable. Application and publishing roles cannot delete templates or their historical versions. Any permanent removal requires a separately authorised maintenance role and a recorded change. Aborting incomplete multipart uploads after seven days is permitted because incomplete uploads are not retained templates.

## Encryption and auditing

Each bucket uses its assigned customer managed KMS key. Writer roles can generate data keys. Reader roles can decrypt only with the relevant key. S3 policies still restrict the accessible object prefixes.

Enable CloudTrail data events for document reads and writes in both buckets. Record changes to bucket policies, access points, lifecycle rules and KMS configuration. Send audit records to `<CENTRAL_AUDIT_DESTINATION>` under `<AUDIT_RETENTION_POLICY>`.

Monitor failed PDF uploads, failed template reads, access denials and unexpected storage growth. Alert on changes to public access settings or retention rules. Route operational alerts to `<OPERATIONS_ALERT_DESTINATION>`.

Logs include the application identity and request ID. They must not contain document contents, credentials or presigned URLs.

## Operations and recovery

Platform Engineering owns bucket configuration, IAM policies, lifecycle controls and encryption. Application teams own template selection, PDF generation, user authorisation and download expiry. `<TEMPLATE_OWNER>` owns template releases and recovery decisions.

Template recovery restores an approved historical version through the recovery role. Regional recovery and independent backup arrangements are recorded in `<TEMPLATE_RECOVERY_RUNBOOK>` with recovery objectives `<TEMPLATE_RPO>` and `<TEMPLATE_RTO>`. Versioning alone is not an independent backup.

Manage both buckets and their policies through `<REPOSITORY_URL>`. Protect buckets and KMS keys from accidental destruction in the deployment workflow. Changes to retention, template deletion or access scope require review through `<CHANGE_PROCESS>`.

## Production verification

| Check | Required result |
|---|---|
| Generate a PDF from an authorised template | PDF saved to the correct shared-events prefix |
| Download through RAP-app | Authorised user receives the correct PDF |
| Request another user's PDF | Rejected by the application |
| Access an unassigned prefix | Rejected by S3 |
| Update a template using an application role | Rejected |
| Delete an object using an application role | Rejected |
| Use an unapproved role or network path | Rejected |
| Request a PDF after 28 days | Rejected by the application |
| Inspect events lifecycle and version state | All PDFs covered by 28-day expiry with no unhandled historical versions |
| Inspect template lifecycle | No automatic expiry for current or historical template versions |
| Restore a template version | Approved version restored and generation verified |
| Review encryption and audit records | Expected KMS keys used and required access events recorded |

Record verification evidence under `<JIRA_REFERENCE>` and link the operational runbook at `<RUNBOOK_URL>`.







