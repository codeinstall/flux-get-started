```
include:
  - project: dwp/engineering/pipeline-solutions/gitlab/templates
    ref: 8.2.10
    file: templates/node-backend-ci-template.yml
  - project: 'dwp/engineering/pipeline-solutions/gitlab/fragments/node-stryker-mutation'
    ref: 3.24.1
    file: 'ci-include-node-stryker-mutation-base.yml'
  - project: dwp/engineering/stage-gate-automation/msa-assessment-tool
    ref: approved
    file: /msa-conftest-fragment.yml
  - component: '${CI_SERVER_FQDN}/dwp/engineering/pipeline-solutions/gitlab/components/version-control/bump-version@0.3.2'
    inputs:
      source_branch: main
      job_stage: '.post'
      create_changelog: true
  - component: $CI_SERVER_FQDN/dwp/engineering/pipeline-solutions/gitlab/components/aws-oidc-secret-retriever/get-secret@2.2.5
    inputs:
      job_name_suffix: nexus-creds-docker-snapshot
      iam_role_arn: arn:aws:iam::339712699350:role/gld-nxr-svc-nx-docker-snap-49ecb911
      secret_name: /svc/nexus/svc-nx-docker-snap-49ecb911/token
      output_file_name: nexus-snapshot-credentials.json
      stage: build
  - component: $CI_SERVER_FQDN/dwp/engineering/pipeline-solutions/gitlab/components/aws-oidc-secret-retriever/get-secret@2.2.5
    inputs:
      job_name_suffix: nexus-creds-docker-release
      iam_role_arn: arn:aws:iam::339712699350:role/gld-nxr-svc-nx-docker-rele-49ecb911
      secret_name: /svc/nexus/svc-nx-docker-rele-49ecb911/token
      output_file_name: nexus-release-credentials.json
      stage: build

```
