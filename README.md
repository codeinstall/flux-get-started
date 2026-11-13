```
default:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  tags: [dwp-aws-x86-linux-l]
  retry:
    max: 2
    when:
      - runner_system_failure

variables:
  ECR_REPOSITORY: '${ACCOUNT_ID_EKS2}.dkr.ecr.eu-west-2.amazonaws.com/pdu-${ENVIRONMENT}/tenant/dmae-raco-data/aa-data-publisher'
  ECR_REPOSITORY_NAME: 'pdu-${ENVIRONMENT}/tenant/dmae-raco-data/aa-data-publisher'
  HCS_EKS_RACO_ROLE_ARN: 'arn:aws:iam::${ACCOUNT_ID_EKS2}:role/eks-pdu-${ENVIRONMENT}-dmae-raco-namespace-admin-role'
  NAMESPACE: 'dmae-raco-data-aa'
  NEXUS_DOCKER_REGISTRY: nexus.nonprod.dwpcloud.uk/docker-snapshots-dwp-digital-channels
  NEXUS_SNAPSHOT_DOCKER_REGISTRY: nexus.nonprod.dwpcloud.uk/docker-snapshots-dwp-digital-channels
  NEXUS_RELEASE_DOCKER_REGISTRY: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-digital-channels
  NEXUS_IMAGE_PATH: digital-channels/report-a-change-once/aa-data-publisher
  SERVICE_NAME: 'aa-data-publisher'
  GENERIC_SERVICE_HELM_CHART_VERSION: '0.70.3'
  IMAGE_TAG: '${CI_COMMIT_SHORT_SHA}'

include:
  - local: '/gitlab-ci/common-jobs.yaml'
  - local: '/gitlab-ci/includes.yaml'
  - local: '/gitlab-ci/stages.yaml'

workflow:
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      variables:
        NEXUS_DOCKER_REGISTRY: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-digital-channels
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_PIPELINE_SOURCE == "web"'
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS'
      when: never
    - if: '$CI_COMMIT_BRANCH'

image-policy:
  stage: image-policy
  extends: .image-policy

get-secret:nexus-creds-docker-release:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - when: never

get-secret:nexus-creds-docker-snapshot:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - when: never

container_scanning:
  needs:
    - job: image-policy
      artifacts: true
    - job: docker-build
      optional: true
      artifacts: true
  allow_failure: false
  variables:
    CS_IMAGE: $SELECTED_NEXUS_DOCKER_REGISTRY/$NEXUS_IMAGE_PATH:$SELECTED_IMAGE_TAG
    SCAN_IMAGE: $SELECTED_NEXUS_DOCKER_REGISTRY/$NEXUS_IMAGE_PATH:$SELECTED_IMAGE_TAG
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - Dockerfile
        - .dockerignore
        - package.json
        - package-lock.json
        - src/**/*
        - '@types/**/*'
        - tsconfig.json
        - .gitlab-ci.yml
        - gitlab-ci/common-jobs.yaml
        - gitlab-ci/includes.yaml
    - when: never

container-scanning-cosign-vuln:
  needs:
    - job: image-policy
      artifacts: true
    - job: docker-build
      optional: true
      artifacts: true
  allow_failure: false
  variables:
    CS_IMAGE: $SELECTED_NEXUS_DOCKER_REGISTRY/$NEXUS_IMAGE_PATH:$SELECTED_IMAGE_TAG
    SCAN_IMAGE: $SELECTED_NEXUS_DOCKER_REGISTRY/$NEXUS_IMAGE_PATH:$SELECTED_IMAGE_TAG
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - Dockerfile
        - .dockerignore
        - package.json
        - package-lock.json
        - src/**/*
        - '@types/**/*'
        - tsconfig.json
        - .gitlab-ci.yml
        - gitlab-ci/common-jobs.yaml
        - gitlab-ci/includes.yaml
    - when: never

openapi-validation:
  rules: [when: never]

apifuzzer_fuzz:
  rules: [when: never]

dast_api:
  rules: [when: never]

node-coverage:
  variables:
    TEST_COMMAND: npm run test:coverage

node-stryker-mutation:
  extends: .stryker-mutation-template
  variables:
    STRYKER_CONFIG_FILE_NAME: ./stryker.config.json
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: always
    - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"'
      when: always
    - when: never

kube-score:
  stage: code-analysis
  extends: .kube-score
  parallel:
    matrix:
      - ENVIRONMENT: dev
      - ENVIRONMENT: test

msa-render:
  extends: .msa-render
  stage: code-analysis
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test]
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
  needs:
    - job: image-policy
      artifacts: true
  allow_failure: true

msa-compliance-check:
  stage: msa-fragment-assessment
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test]
  needs:
    - job: msa-render
      artifacts: true
  variables:
    MSA_TARGET_DIR: "msa-rendered/${ENVIRONMENT}"
    MSA_FAIL_ON_WARN: "false"
  before_script:
    - |
      set -eu

      if [ ! -s "msa-rendered/${ENVIRONMENT}/app.yaml" ]; then
        echo "MSA assessment cannot run: rendered manifests missing or empty for ${ENVIRONMENT}."
        exit 1
      fi
  allow_failure: true

msa-slack-notify:
  rules:
    - when: never

docker-build:
  stage: build
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/docker-cosign:1.11.1@sha256:57409822a66b9119165fabeeba85c2c42e23e994aa0f63093dbeec448f9a13b9
  tags:
    - dwp-aws-x86-linux-l-dind
  services:
    - name: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/docker-cosign:1.11.1@sha256:57409822a66b9119165fabeeba85c2c42e23e994aa0f63093dbeec448f9a13b9
      alias: docker
  needs:
    - job: image-policy
      artifacts: true
    - job: get-secret:nexus-creds-docker-snapshot
      optional: true
      artifacts: true
    - job: get-secret:nexus-creds-docker-release
      optional: true
      artifacts: true
  extends:
    - .docker-build-and-push-to-nexus
  variables:
    DOCKER_TLS_CERTDIR: '/certs'
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: on_success
    - when: never

virus-scan:
  stage: security-dynamic-analysis
  needs: []
  variables:
    CLAM_SCAN_ARGS: '--exclude=(^|/)\.npm(/|$) --exclude=(^|/)node_modules(/|$) --exclude-dir=(^|/)\.npm(/|$) --exclude-dir=(^|/)node_modules(/|$)'

# ----------------------------------------------------------------------------------------------------------------
# DEPLOY DEV
# ----------------------------------------------------------------------------------------------------------------

push-to-ecr-pdu-dev:
  stage: deploy-dev
  needs:
    - job: image-policy
      artifacts: true
    - job: docker-build
      optional: true
      artifacts: true
    - job: container_scanning
      optional: true
    - job: container-scanning-cosign-vuln
      optional: true
  extends:
    - .assume-hcs-eks-raco-role
    - .push-to-ecr
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'dev'
  rules:
    - when: on_success

ecr-vuln-scan-pdu-dev:
  stage: deploy-dev
  needs:
    - job: image-policy
      artifacts: true
    - job: push-to-ecr-pdu-dev
      artifacts: true
  extends:
    - .assume-hcs-eks-raco-role
    - .scan-ecr-vulns
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'dev'
  retry:
    max: 2
    exit_codes: 255

deploy-helm-values-pdu-dev:
  stage: deploy-dev
  needs:
    - job: image-policy
      artifacts: true
    - job: ecr-vuln-scan-pdu-dev
  extends:
    - .assume-hcs-eks-raco-role
    - .helm-upgrade
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'dev'

dev-version-check:
  stage: deploy-dev
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/kube-score-helm-yq:1.1.0@sha256:3c293b09b0b57031d2a16247b44923150b25180b8aa0b1fc7dbf48df1b3b9fcf
  needs:
    - job: image-policy
      artifacts: true
    - job: deploy-helm-values-pdu-dev
  when: on_success
  allow_failure: false
  script:
    - |
      set -eu
      if [ "${DEPLOYMENT_PURPOSE}" != "application-pr" ]; then
        echo "Not an application-change PR - Dev version check skipped."
        exit 0
      fi
      if [ "${DEV_TAG_CHANGED}" != "true" ]; then
        echo "Application changes require a new semantic version in Dev values."
        exit 1
      fi

# ----------------------------------------------------------------------------------------------------------------
# DEPLOY TEST
# ----------------------------------------------------------------------------------------------------------------

push-to-ecr-pdu-test:
  stage: deploy-test
  needs:
    - job: image-policy
      artifacts: true
    - job: deploy-helm-values-pdu-dev
    - job: docker-build
      optional: true
      artifacts: true
  extends:
    - .assume-hcs-eks-raco-role
    - .push-to-ecr
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'test'
  rules:
    - when: manual
      allow_failure: true

ecr-vuln-scan-pdu-test:
  stage: deploy-test
  needs:
    - job: image-policy
      artifacts: true
    - job: push-to-ecr-pdu-test
      artifacts: true
  extends:
    - .assume-hcs-eks-raco-role
    - .scan-ecr-vulns
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'test'
  retry:
    max: 2
    exit_codes: 255

deploy-helm-values-pdu-test:
  stage: deploy-test
  extends:
    - .assume-hcs-eks-raco-role
    - .helm-upgrade
  needs:
    - job: image-policy
      artifacts: true
    - job: ecr-vuln-scan-pdu-test
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
    OIDC_ROLE_ACCOUNT_ID: '533267201269'
    OIDC_ROLE_NAME: 'raco-oidc-runner-np'
    ENVIRONMENT: 'test'

test-version-check:
  stage: deploy-test
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/kube-score-helm-yq:1.1.0@sha256:3c293b09b0b57031d2a16247b44923150b25180b8aa0b1fc7dbf48df1b3b9fcf
  needs:
    - job: image-policy
      artifacts: true
    - job: deploy-helm-values-pdu-test
  when: on_success
  allow_failure: false
  script:
    - |
      set -eu
      if [ "${DEPLOYMENT_PURPOSE}" != "application-pr" ]; then
        echo "Not an application-change PR - Test version check skipped."
        exit 0
      fi
      if [ "${DEV_TAG_CHANGED}" != "true" ]; then
        echo "Application changes require a new semantic version in Dev values."
        exit 1
      fi
      if [ "${TEST_IMAGE_TAG}" != "${INTENDED_VERSION}" ]; then
        echo "Test values must use ${INTENDED_VERSION}."
        exit 1
      fi

# ----------------------------------------------------------------------------------------------------------------
# DEPLOY STAGING
# ----------------------------------------------------------------------------------------------------------------

push-to-ecr-pdu-stage:
  stage: deploy-stage
  needs: [deploy-helm-values-pdu-test]
  environment:
    name: staging
  extends:
    - .assume-hcs-eks-raco-role
    - .push-to-ecr
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'stage'
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: manual
    - when: never

ecr-vuln-scan-pdu-stage:
  stage: deploy-stage
  needs: [push-to-ecr-pdu-stage]
  environment:
    name: staging
  extends:
    - .assume-hcs-eks-raco-role
    - .scan-ecr-vulns
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'stage'
  retry:
    max: 2
    exit_codes: 255
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: on_success
    - when: never

deploy-helm-values-pdu-stage:
  stage: deploy-stage
  needs: [ecr-vuln-scan-pdu-stage]
  environment:
    name: staging
  extends:
    - .assume-hcs-eks-raco-role
    - .helm-upgrade
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'stage'
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: manual
    - when: never

# ----------------------------------------------------------------------------------------------------------------
# DEPLOY PROD
# ----------------------------------------------------------------------------------------------------------------

push-to-ecr-pdu-prod:
  stage: deploy-prod
  needs: [deploy-helm-values-pdu-stage]
  environment:
    name: production
  extends:
    - .assume-hcs-eks-raco-role
    - .push-to-ecr
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'prod'
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: manual
    - when: never

ecr-vuln-scan-pdu-prod:
  stage: deploy-prod
  needs: [push-to-ecr-pdu-prod]
  environment:
    name: production
  extends:
    - .assume-hcs-eks-raco-role
    - .scan-ecr-vulns
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'prod'
  retry:
    max: 2
    exit_codes: 255
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: on_success
    - when: never

deploy-helm-values-pdu-prod:
  stage: deploy-prod
  needs: [ecr-vuln-scan-pdu-prod]
  environment:
    name: production
  extends:
    - .assume-hcs-eks-raco-role
    - .helm-upgrade
  variables:
    ACCOUNT_ID_EKS2: '726642692718'
    OIDC_ROLE_ACCOUNT_ID: '366305978915'
    OIDC_ROLE_NAME: 'raco-oidc-runner-prod'
    ENVIRONMENT: 'prod'
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main" || $CI_COMMIT_TAG =~ /^release\\//'
      when: manual
    - when: never

```
