Add msa-fragment-assessment after code-analysis. We can use a separate render job so the existing kube-score job stays unchanged.

1. Update gitlab-ci/stages.yaml:
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

2. Add the fragment to the existing include list in .gitlab-ci.yml:
  - project: 'dwp/engineering/stage-gate-automation/msa-assessment-tool'
    ref: approved
    file: '/msa-conftest-fragment.yml'

3. Add these jobs to .gitlab-ci.yml:
# Render the chart for assessment without deploying it.
msa-render:
  stage: code-analysis
  extends: .kube-score
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test]
  variables:
    ACCOUNT_ID_EKS2: ''
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
  allow_failure: true

# Run the supplied MSA checks separately for each environment.
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
  allow_failure: true

# Disable the fragment's optional notification job for now.
msa-slack-notify:
  rules:
    - when: never

The render job reuses .kube-score’s tooling image but replaces its script. Both rendered environment folders are downloaded by each MSA job, but each job assesses only its own environment directory.

This gives you:
- Separate dev and test assessments.
- The chart version, release name, namespace and image settings from your pipeline.
- A conftest-report.json artifact in each assessment job.
- Non-blocking MSA jobs during initial integration.
Before merging: validate the combined configuration in GitLab CI Lint and run a branch pipeline. Check that both assessments actually process manifests rather than skip. The runner needs access to the fragment’s CI image and its job token must be able to clone the policy repository.

This still uses the GitLab chart package, so confirm it matches the same-version chart deployed from ECR. Staging and Production assessment configurations are not included in this initial dev/test setup.
