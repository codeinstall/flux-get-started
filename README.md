```
# Render the chart for assessment without deploying it.
msa-render:
  stage: code-analysis
  extends: .kube-score
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test]
  variables:
    ACCOUNT_ID_EKS2: '943009210227'
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
```
