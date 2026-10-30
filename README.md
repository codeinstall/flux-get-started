```
Move the rendering implementation into gitlab-ci/common-jobs.yaml and extend that new template from .gitlab-ci.yml. It uses the GitLab tooling image directly, with no extension from kube-score.

1. Add this to gitlab-ci/common-jobs.yaml:
.msa-render:
  image: $CI_REGISTRY/dwp/engineering/images/ks8-tools:1.3.0@sha256:2b18ba539920bfb922c13f8b72aa8c31f015ffbb7eb0bf407b00df47109a8de3
  script:
    - |
      set -eu

      helm repo add \
        --username gitlab-ci-token \
        --password "${CI_JOB_TOKEN}" \
        hcs-helm-charts \
        "${CI_API_V4_URL}/projects/3913/packages/helm/stable"

      helm repo update

      mkdir -p "msa-rendered/${ENVIRONMENT}"

      helm template "${SERVICE_NAME}" \
        hcs-helm-charts/generic-service \
        --version "${GENERIC_SERVICE_HELM_CHART_VERSION}" \
        --namespace "${NAMESPACE}" \
        --values "values/pdu-${ENVIRONMENT}-values.yaml" \
        --set-string deployment.containers.main.image.tag="${IMAGE_TAG}" \
        --set-string deployment.containers.main.image.repository="${ECR_REPOSITORY}" \
        > "msa-rendered/${ENVIRONMENT}/app.yaml"

      test -s "msa-rendered/${ENVIRONMENT}/app.yaml"
  artifacts:
    paths:
      - msa-rendered/
    expire_in: 7 days

2. Replace the existing MSA jobs in .gitlab-ci.yml with:
# Render the chart separately for dev and test.
msa-render:
  extends: .msa-render
  stage: code-analysis
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test]
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
  allow_failure: true

# Assess each environment's rendered manifests.
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

# Disable optional Slack notifications.
msa-slack-notify:
  rules:
    - when: never

The additional before_script makes missing render output visible as an allowed job failure. Otherwise, the fragment can skip assessment and exit successfully when the target directory is missing.

3. Keep this entry under the existing include: in gitlab-ci/includes.yaml:
  - project: dwp/engineering/stage-gate-automation/msa-assessment-tool
    ref: approved
    file: /msa-conftest-fragment.yml

4. Keep the assessment stage after code-analysis in gitlab-ci/stages.yaml:
stages:
  - unit-test
  - code-test
  - code-analysis
  - msa-fragment-assessment
  - build
  - security-static-analysis
  - security-dynamic-analysis
  - install
  - publish
  - deploy-dev
  - deploy-test
  - deploy-stage
  - deploy-prod
  - integration-test

Your existing .kube-score template and kube-score job stay unchanged. If the include and stage are already committed, this refactor changes only common-jobs.yaml and .gitlab-ci.yml.

Commit message:
ci: move MSA rendering into a dedicated common template
```
