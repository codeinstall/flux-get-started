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














# S3 Storage Design for Three Applications Across Two Buckets

This document defines the proposed S3 storage configuration for Atlas Portal, Beacon Processor and Cedar Reports. It records where data is stored, how each application accesses it, the permissions assigned to each role and how data is retained and removed.

**Template status:** All application names, resource identifiers, owners and retention periods below are fictional examples. They describe a proposed configuration, not deployed resources. Replace them and approve the outstanding decisions before implementation.

## Document control

| Field | Value |
|---|---|
| Status | Draft for technical review |
| Version | 0.2 |
| Environment | Production example |
| Document owner | Platform Engineering |
| Service owner | Example Digital Services |
| Technical approver | Technical Design Authority |
| Security reviewer | Cloud Security Team |
| Data retention approver | Data Owner |
| Jira epic | CLOUD-1000 |
| Implementation ticket | CLOUD-1001 |
| Confluence parent page | Cloud Platform / Storage |
| Review frequency | Every six months and after material changes |

## 1 Scope and design

Use two S3 general purpose buckets. Bucket A holds Atlas Portal data under `atlas/` and Beacon Processor data under `beacon/`. Bucket B holds Cedar Reports data under `cedar/`. This application-to-bucket mapping is illustrative and can be replaced with the actual mapping. Each application receives its own S3 access point and IAM role. Application access is limited to its assigned prefix. Cross-application access is excluded from this example and requires a separate approved permission change.

This example assumes the workloads run in an AWS VPC in the same account and region as both buckets. Development and test use separate resources and roles. If the workloads run in an external OpenShift cluster, confirm connectivity and workload identity before adopting the VPC-only access model.

Prefixes provide logical separation between Atlas and Beacon within Bucket A. Bucket-level settings and administrative changes to Bucket A affect both applications. Cedar uses Bucket B with separate bucket policies, lifecycle configuration and encryption key. Revisit the Atlas and Beacon sharing decision if they need different ownership, isolation or recovery requirements.

## 2 Bucket configuration

| Setting | Bucket A | Bucket B |
|---|---|---|
| Purpose | Application uploads and processing data | Reports and exports |
| Applications | Atlas Portal and Beacon Processor | Cedar Reports |
| Bucket name | `example-digital-prod-app-data-123456789012` | `example-digital-prod-reports-123456789012` |
| Bucket ARN | `arn:aws:s3:::example-digital-prod-app-data-123456789012` | `arn:aws:s3:::example-digital-prod-reports-123456789012` |
| Region | `eu-west-2` / London | `eu-west-2` / London |
| Prefixes | `atlas/` and `beacon/` | Bucket B / `cedar/` |
| Access points | `atlas-prod-ap` and `beacon-prod-ap` | `cedar-prod-ap` |
| KMS key alias | `alias/example-prod-app-data` | `alias/example-prod-reports` |
| KMS key ARN | `<BUCKET_A_KMS_KEY_ARN>` | `<BUCKET_B_KMS_KEY_ARN>` |

The following baseline applies independently to both buckets.

| Setting | Example value or required configuration |
|---|---|
| AWS account name | example-digital-prod |
| AWS account ID | `123456789012` |
| Region | `eu-west-2` / London |
| Initial storage class | S3 Standard |
| Versioning | Enabled |
| Encryption | SSE-KMS with a separate customer managed key for each bucket |
| Public access | Block all public access at account, bucket and access point levels |
| Object ownership | Bucket owner enforced with ACLs disabled |
| Transport | HTTPS required |
| VPC | `<PRODUCTION_VPC_ID>` |
| S3 VPC endpoint | `<S3_VPC_ENDPOINT_ID>` |
| Replication | Not configured in this example; recovery requirement TBC |
| Object Lock | Not configured in this example; immutable retention requirement TBC |
| Infrastructure code | `<REPOSITORY_URL>` / `terraform/storage/` |

Required resource tags: `Environment=prod`, `Service=example-digital`, `Owner=platform-engineering`, `CostCentre=CC1000`, `DataClassification=internal-example` and `ManagedBy=terraform`. The classification must be confirmed against the actual data.

## 3 Application access mapping

| Application | Bucket | Data stored | Prefix | S3 access point | IAM role | Access |
|---|---|---|---|---|---|---|
| Atlas Portal | Bucket A | User uploads and supporting files | Bucket A / `atlas/` | `atlas-prod-ap` | `atlas-prod-s3-role` | List, read and write |
| Beacon Processor | Bucket A | Processing inputs and outputs | Bucket A / `beacon/` | `beacon-prod-ap` | `beacon-prod-s3-role` | List, read, write and delete |
| Cedar Reports | Bucket B | Generated reports and exports | Bucket B / `cedar/` | `cedar-prod-ap` | `cedar-prod-s3-role` | List, read and write |

Bucket A and Bucket B refer to the exact bucket names in section 2. Each access point is attached to its application's assigned bucket. No application can list or access another application's prefix or the other bucket. Atlas and Cedar rely on lifecycle rules for scheduled deletion. Beacon can delete current objects because its example workflow includes temporary processing data.

### Access point identifiers

| Application | Access point ARN | Network origin |
|---|---|---|
| Atlas Portal | `arn:aws:s3:eu-west-2:123456789012:accesspoint/atlas-prod-ap` | VPC / `<PRODUCTION_VPC_ID>` |
| Beacon Processor | `arn:aws:s3:eu-west-2:123456789012:accesspoint/beacon-prod-ap` | VPC / `<PRODUCTION_VPC_ID>` |
| Cedar Reports | `arn:aws:s3:eu-west-2:123456789012:accesspoint/cedar-prod-ap` | VPC / `<PRODUCTION_VPC_ID>` |

Applications use their access point ARN or the AWS-generated access point alias in the SDK bucket parameter where supported. Record the generated aliases in deployment configuration. Do not construct an alias manually. Configure the AWS region as `eu-west-2` and confirm SDK support during integration testing.

An S3 access point provides an application-specific policy boundary. The VPC endpoint provides the network route to S3. Both must be configured for the intended access path.

## 4 IAM roles and permissions

Role ARNs follow `arn:aws:iam::123456789012:role/<ROLE_NAME>` using the names in section 3.

| Permission | Atlas role | Beacon role | Cedar role | Scope |
|---|---|---|---|---|
| `s3:ListBucket` | Allow | Allow | Allow | Own access point with an `s3:prefix` condition |
| `s3:GetObject` | Allow | Allow | Allow | Own prefix through own access point |
| `s3:PutObject` | Allow | Allow | Allow | Own prefix through own access point |
| `s3:DeleteObject` | No grant | Allow | No grant | Beacon prefix only |
| `s3:AbortMultipartUpload` | Allow | Allow | Allow | Own prefix only |
| `s3:ListMultipartUploadParts` | Allow | Allow | Allow | Own prefix only |
| `s3:GetObjectVersion` | No grant | No grant | No grant | Recovery role only |
| `s3:DeleteObjectVersion` | No grant | No grant | No grant | Approved maintenance role only |
| Bucket, lifecycle and access point administration | No grant | No grant | No grant | Infrastructure deployment role |
| `kms:GenerateDataKey` | Allow | Allow | Allow | Assigned bucket KMS key only |
| `kms:Decrypt` | Allow | Allow | Allow | Assigned bucket KMS key only |

For object actions, scope resources to the access point object ARN, for example `arn:aws:s3:eu-west-2:123456789012:accesspoint/atlas-prod-ap/object/atlas/*`. Scope listing to the access point ARN and limit requested prefixes to `atlas/` and `atlas/*`. Apply the equivalent restrictions for Beacon and Cedar.

“No grant” describes the intended effective access. Review all attached policies, resource policies and permission boundaries to ensure another policy does not grant the action.

### Workload identity and trust

Use temporary credentials through the approved workload identity integration. Do not place long-lived AWS access keys in application configuration. For an OIDC-based integration, restrict each role trust policy to the approved issuer, audience and exact workload subject.

| Application | Example workload subject | Identity provider |
|---|---|---|
| Atlas Portal | `system:serviceaccount:atlas-prod:atlas-portal` | `<OIDC_PROVIDER_ARN>` |
| Beacon Processor | `system:serviceaccount:beacon-prod:beacon-worker` | `<OIDC_PROVIDER_ARN>` |
| Cedar Reports | `system:serviceaccount:cedar-prod:cedar-reports` | `<OIDC_PROVIDER_ARN>` |

Use the trust policy required by the selected runtime integration. Confirm credential acquisition and rotation before granting data access.

### Policy responsibilities

The role policy, access point policy, bucket policy, endpoint policy and KMS key policy must support the same intended request. Bucket A must permit the Atlas and Beacon access points. Bucket B must permit the Cedar access point. Apply the appropriate bucket policy separately to each bucket. Access point restrictions alone do not block direct bucket access, so separately restrict application data access to the intended path. Preserve explicitly approved service and operational access. See [AWS access point policy guidance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-policies.html).

Grant Atlas and Beacon KMS permissions on `<BUCKET_A_KMS_KEY_ARN>` only. Grant Cedar KMS permissions on `<BUCKET_B_KMS_KEY_ARN>` only. Restrict each key to the approved S3 service context. Validate any encryption-context conditions against the chosen S3 Bucket Key setting. Encryption permissions do not replace S3 prefix restrictions.

### Operational roles

| Role | Purpose | Boundary |
|---|---|---|
| `storage-deploy-role` | Manage both buckets, access points and lifecycle through infrastructure code | Approved pipeline; no routine object access |
| `storage-recovery-role` | Read previous versions and restore approved objects | Temporary access to the affected prefix with a recorded ticket |
| `storage-maintenance-role` | Permanently remove approved object versions | Time-limited access and explicit approval |
| `storage-audit-role` | Review configuration and audit records | No application payload access |

## 5 Retention and lifecycle

**The periods below are illustrative business values.** They are not legal requirements or an approved retention schedule. The Data Owner must approve each period before the rules are enabled.

| Bucket and data scope | Current object expiry | Noncurrent version expiry | Storage transition | Owner |
|---|---|---|---|---|
| Bucket A / `atlas/` | 365 days after creation | 30 days after becoming noncurrent | None in initial example | Atlas Product Owner |
| Bucket A / `beacon/` | 90 days after creation | 7 days after becoming noncurrent | None in initial example | Beacon Service Owner |
| Bucket B / `cedar/` | 180 days after creation | 30 days after becoming noncurrent | None in initial example | Cedar Product Owner |

Configure Atlas and Beacon lifecycle rules on Bucket A and Cedar lifecycle rules on Bucket B. Apply prefix-filtered lifecycle rules independently in each bucket. Abort incomplete multipart uploads after seven days. Configure removal of expired delete markers where appropriate.

In a versioned bucket, current-version expiration normally creates a delete marker. Previous versions require their own expiry rule. Noncurrent retention is measured from when a version becomes noncurrent, so the example periods do not represent a strict maximum age for every stored byte. Lifecycle deletion is asynchronous. See [AWS lifecycle configuration guidance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html).

An application delete permission does not enforce minimum retention. If data must be protected against early deletion or overwriting, define and approve an Object Lock design separately. Backups, replicas and exported copies need their own approved retention rules.

## 6 Monitoring and audit

Enable CloudTrail S3 data events for both buckets and the required application object operations and record configuration changes. Send audit records to a separate central logging destination with access and retention controlled by the security team. Example audit retention is 365 days, subject to approval.

Monitor access-denied responses, unusual delete activity, bucket growth, request failures and changes to public access, encryption or lifecycle configuration. Record the alert destination as `<OPERATIONS_ALERT_CHANNEL>` and the dashboard as `<STORAGE_DASHBOARD_URL>`.

Application logs should include request IDs and operation outcomes without recording credentials or sensitive object contents. Platform Engineering owns storage alerts. Application teams own failed business operations and retries.

## 7 Recovery and support

Versioning supports recovery from accidental changes within the retained version window. It does not provide an independent backup or regional recovery plan. The service owner must confirm whether separate backups or replication are required.

| Recovery requirement | Placeholder |
|---|---|
| Recovery point objective | `<APPROVED_RPO>` |
| Recovery time objective | `<APPROVED_RTO>` |
| Backup or replication decision | `<DECISION_AND_TICKET>` |
| Recovery runbook | `<RUNBOOK_URL>` |
| Escalation contact | `<PLATFORM_SUPPORT_CONTACT>` |

For object recovery, identify the affected bucket, key and version, request temporary recovery access, restore the approved version and ask the application owner to validate the result. Attach the outcome to the incident ticket. Complete a restore test before production approval.

## 8 Validation and acceptance

| Test | Expected result | Evidence |
|---|---|---|
| Each application assumes its own role | Temporary credentials issued only to the approved workload | `<TEST_LINK>` |
| List, upload and read within own prefix | Successful through the assigned access point | `<TEST_LINK>` |
| Access another application's prefix or access point | Denied | `<TEST_LINK>` |
| Access the other bucket or its KMS key as an application | Denied | `<TEST_LINK>` |
| Direct bucket access by application roles | Denied by the configured access boundary | `<TEST_LINK>` |
| Delete as Atlas or Cedar | Denied | `<TEST_LINK>` |
| Delete as Beacon | Allowed only within `beacon/` | `<TEST_LINK>` |
| Permanently delete a version as an application | Denied | `<TEST_LINK>` |
| Upload and retrieve encrypted objects | Correct KMS key used; unauthorised key use denied | `<TEST_LINK>` |
| Public or unapproved network access | Denied | `<TEST_LINK>` |
| Multipart upload and abort | Successful within the assigned prefix | `<TEST_LINK>` |
| Lifecycle configuration | Prefixes and current/noncurrent rules match approved retention | `<TEST_LINK>` |
| Audit trail and alerts | Required operations recorded and test alert received | `<TEST_LINK>` |
| Restore a previous object version | Application owner confirms recovery | `<TEST_LINK>` |

## 9 Implementation and change control

Track delivery under `CLOUD-1001` with linked tasks for resource creation, workload identity, policy review, application configuration, retention approval and validation evidence. Store all resource definitions and policies in the infrastructure repository.

Deploy and test in a non-production environment first. Review the infrastructure plan before production changes. Treat deletion, lifecycle shortening and KMS key removal as separate changes requiring impact review. Protect both buckets and their keys from accidental destruction in the deployment workflow.

Rollback should restore the last approved policy and application configuration. Reverting infrastructure code cannot recover permanently deleted data. Confirm the recovery path before enabling destructive lifecycle changes.

## 10 Outstanding decisions and approvals

| Item | Responsible owner | Status |
|---|---|---|
| Replace account, bucket, access point and role placeholders | Platform Engineering | TBC |
| Confirm the two-bucket mapping and Atlas and Beacon prefix isolation | Technical Design Authority | TBC |
| Confirm workload location, identity and network path | Platform and Application Teams | TBC |
| Approve permissions including Beacon deletion | Application Owners and Security | TBC |
| Confirm classification and retention periods | Data Owner | TBC |
| Confirm KMS ownership and key recovery arrangements | Cloud Security Team | TBC |
| Agree backup, replication, RPO and RTO | Service Owner | TBC |
| Complete validation and attach evidence | Platform Engineering | TBC |
| Approve production implementation | Change Owner | TBC |

