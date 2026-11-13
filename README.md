```
.assume-hcs-eks-raco-role: &assume-hcs-eks-raco-role
  before_script:
    - export AWS_OIDC_DEPLOYER_ROLE=arn:aws:iam::${OIDC_ROLE_ACCOUNT_ID}:role/${OIDC_ROLE_NAME}
    - echo "Assuming OIDC Deployer role - ${AWS_OIDC_DEPLOYER_ROLE}"
    - export AWS_STS_Regional_Endpoints=regional
    - >
      export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s"
      $(aws sts assume-role-with-web-identity
      --role-arn ${AWS_OIDC_DEPLOYER_ROLE}
      --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
      --web-identity-token ${GITLAB_OIDC_TOKEN}
      --duration-seconds 3600
      --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
      --output text))
    - aws sts get-caller-identity
    - echo "Assuming HCS EKS RACO namespace role"
    - >
      export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s"
      $(aws sts assume-role
      --role-arn ${HCS_EKS_RACO_ROLE_ARN}
      --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
      --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
      --output text))
    - aws sts get-caller-identity

.image-policy: &image-policy
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/kube-score-helm-yq:1.1.0@sha256:3c293b09b0b57031d2a16247b44923150b25180b8aa0b1fc7dbf48df1b3b9fcf
  script:
    - |
      set -eu

      read_tag() {
        yq eval -r '.deployment.containers.main.image.tag // ""' "$1"
      }

      read_base_tag() {
        git show "${COMPARE_SHA}:$1" | yq eval -r '.deployment.containers.main.image.tag // ""' -
      }

      DEV_IMAGE_TAG=$(read_tag values/pdu-dev-values.yaml)
      TEST_IMAGE_TAG=$(read_tag values/pdu-test-values.yaml)

      for IMAGE_TAG in "${DEV_IMAGE_TAG}" "${TEST_IMAGE_TAG}"; do
        if ! printf '%s\n' "${IMAGE_TAG}" | grep -Eq '^v(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)$'; then
          echo "Dev and Test image tags must use vMAJOR.MINOR.PATCH."
          exit 1
        fi
      done

      APP_CHANGED=false
      DEV_TAG_CHANGED=false
      DEPLOYMENT_PURPOSE=committed
      INTENDED_VERSION=${DEV_IMAGE_TAG}
      SELECTED_IMAGE_TAG=${DEV_IMAGE_TAG}
      COMPARE_SHA=

      if [ "${CI_PIPELINE_SOURCE}" = "merge_request_event" ]; then
        COMPARE_SHA=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-}
        if [ -z "${COMPARE_SHA}" ]; then
          echo "The merge request comparison commit is not available."
          exit 1
        fi
        if ! git cat-file -e "${COMPARE_SHA}^{commit}" 2>/dev/null; then
          echo "Comparison commit ${COMPARE_SHA} is missing from the checkout."
          exit 1
        fi

        CHANGED_FILES=$(git diff --name-only "${COMPARE_SHA}" "${CI_COMMIT_SHA}")
        if printf '%s\n' "${CHANGED_FILES}" | grep -Eq '^(Dockerfile|\.dockerignore|package(-lock)?\.json|src/|@types/|tsconfig\.json|\.gitlab-ci\.yml|gitlab-ci/common-jobs\.yaml|gitlab-ci/includes\.yaml)'; then
          APP_CHANGED=true
        fi

        BASE_DEV_IMAGE_TAG=$(read_base_tag values/pdu-dev-values.yaml || true)
        if [ "${DEV_IMAGE_TAG}" != "${BASE_DEV_IMAGE_TAG}" ]; then
          DEV_TAG_CHANGED=true
        fi

        if [ "${APP_CHANGED}" = "true" ]; then
          DEPLOYMENT_PURPOSE=application-pr
          if [ "${DEV_TAG_CHANGED}" != "true" ]; then
            SELECTED_IMAGE_TAG="${DEV_IMAGE_TAG}-${CI_COMMIT_SHORT_SHA}"
          fi
        fi
      fi

      if ! printf '%s\n' "${SELECTED_IMAGE_TAG}" | grep -Eq '^v(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-[0-9a-f]+)?$'; then
        echo "Selected image tag ${SELECTED_IMAGE_TAG} is not valid."
        exit 1
      fi

      if [ "${DEPLOYMENT_PURPOSE}" = "application-pr" ]; then
        SELECTED_NEXUS_DOCKER_REGISTRY=${NEXUS_SNAPSHOT_DOCKER_REGISTRY}
        SELECTED_NEXUS_CREDENTIAL_FILE=nexus-snapshot-credentials.json
      else
        SELECTED_NEXUS_DOCKER_REGISTRY=${NEXUS_RELEASE_DOCKER_REGISTRY}
        SELECTED_NEXUS_CREDENTIAL_FILE=nexus-release-credentials.json
      fi

      BUILD_INPUTS_DIGEST=$(git ls-tree -r "${CI_COMMIT_SHA}" -- \
        Dockerfile .dockerignore package.json package-lock.json src @types tsconfig.json \
        .gitlab-ci.yml gitlab-ci/common-jobs.yaml gitlab-ci/includes.yaml \
        | git hash-object --stdin)

      {
        echo "APP_CHANGED=${APP_CHANGED}"
        echo "DEV_TAG_CHANGED=${DEV_TAG_CHANGED}"
        echo "DEPLOYMENT_PURPOSE=${DEPLOYMENT_PURPOSE}"
        echo "INTENDED_VERSION=${INTENDED_VERSION}"
        echo "SELECTED_IMAGE_TAG=${SELECTED_IMAGE_TAG}"
        echo "DEV_IMAGE_TAG=${DEV_IMAGE_TAG}"
        echo "TEST_IMAGE_TAG=${TEST_IMAGE_TAG}"
        echo "BUILD_INPUTS_DIGEST=${BUILD_INPUTS_DIGEST}"
        echo "SELECTED_NEXUS_DOCKER_REGISTRY=${SELECTED_NEXUS_DOCKER_REGISTRY}"
        echo "SELECTED_NEXUS_CREDENTIAL_FILE=${SELECTED_NEXUS_CREDENTIAL_FILE}"
      } | tee image-policy.env
  artifacts:
    reports:
      dotenv: image-policy.env
    paths:
      - image-policy.env

.promote-to-nexus-release:
  image: $CI_REGISTRY/dwp/engineering/images/ks8-tools:1.3.0@sha256:2b18ba539920bfb922c13f8b72aa8c31f015ffbb7eb0bf407b00df47109a8de3
  before_script:
    - export RELEASE_USER=$(yq eval -r '.username' nexus-release-credentials.json)
    - export RELEASE_TOKEN=$(yq eval -r '.token' nexus-release-credentials.json)
    - echo "${RELEASE_TOKEN}" | skopeo login --username "${RELEASE_USER}" --password-stdin "${NEXUS_RELEASE_DOCKER_REGISTRY}"
  script:
    - |
      set -eu

      IMAGE_TAG=${DEV_IMAGE_TAG}
      SNAPSHOT_IMAGE="${NEXUS_SNAPSHOT_DOCKER_REGISTRY}/${NEXUS_IMAGE_PATH}:${IMAGE_TAG}"
      RELEASE_IMAGE="${NEXUS_RELEASE_DOCKER_REGISTRY}/${NEXUS_IMAGE_PATH}:${IMAGE_TAG}"
      SNAPSHOT_DIGEST=
      RELEASE_DIGEST=

      set +e
      INSPECT_OUTPUT=$(skopeo inspect --format '{{.Digest}}' "docker://${RELEASE_IMAGE}" 2>&1)
      INSPECT_STATUS=$?
      set -e
      if [ "${INSPECT_STATUS}" -eq 0 ]; then
        RELEASE_DIGEST=$(printf '%s\n' "${INSPECT_OUTPUT}" | tail -n 1)
        if ! printf '%s\n' "${RELEASE_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
          echo "Nexus returned an invalid image digest for ${RELEASE_IMAGE}."
          exit 1
        fi
      elif ! printf '%s\n' "${INSPECT_OUTPUT}" | grep -Eqi 'manifest unknown|name unknown|manifest .*not found|manifest does not exist'; then
        printf '%s\n' "${INSPECT_OUTPUT}"
        exit "${INSPECT_STATUS}"
      fi

      set +e
      INSPECT_OUTPUT=$(skopeo inspect --format '{{.Digest}}' "docker://${SNAPSHOT_IMAGE}" 2>&1)
      INSPECT_STATUS=$?
      set -e
      if [ "${INSPECT_STATUS}" -eq 0 ]; then
        SNAPSHOT_DIGEST=$(printf '%s\n' "${INSPECT_OUTPUT}" | tail -n 1)
        if ! printf '%s\n' "${SNAPSHOT_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
          echo "Nexus returned an invalid image digest for ${SNAPSHOT_IMAGE}."
          exit 1
        fi
      elif ! printf '%s\n' "${INSPECT_OUTPUT}" | grep -Eqi 'manifest unknown|name unknown|manifest .*not found|manifest does not exist'; then
        printf '%s\n' "${INSPECT_OUTPUT}"
        exit "${INSPECT_STATUS}"
      fi

      if [ -n "${RELEASE_DIGEST}" ]; then
        if [ -n "${SNAPSHOT_DIGEST}" ] && [ "${RELEASE_DIGEST}" != "${SNAPSHOT_DIGEST}" ]; then
          echo "Release image ${IMAGE_TAG} exists with a different digest."
          exit 1
        fi
        echo "Release image ${IMAGE_TAG} already exists with digest ${RELEASE_DIGEST}."
        exit 0
      fi

      if [ -z "${SNAPSHOT_DIGEST}" ]; then
        echo "Snapshot image ${IMAGE_TAG} does not exist and cannot be promoted."
        exit 1
      fi

      skopeo copy --override-arch=amd64 --override-os=linux "docker://${SNAPSHOT_IMAGE}" "docker://${RELEASE_IMAGE}"
      RELEASE_DIGEST=$(skopeo inspect --format '{{.Digest}}' "docker://${RELEASE_IMAGE}")
      if ! printf '%s\n' "${RELEASE_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
        echo "Nexus returned an invalid image digest for ${RELEASE_IMAGE}."
        exit 1
      fi
      if [ "${SNAPSHOT_DIGEST}" != "${RELEASE_DIGEST}" ]; then
        echo "Snapshot and release image digests do not match."
        exit 1
      fi
  after_script:
    - skopeo logout "${NEXUS_RELEASE_DOCKER_REGISTRY}"

.helm-upgrade: &helm-upgrade
  image: $CI_REGISTRY/dwp/engineering/images/ks8-tools:1.3.0@sha256:2b18ba539920bfb922c13f8b72aa8c31f015ffbb7eb0bf407b00df47109a8de3
  script:
    - aws ecr get-login-password --region eu-west-2 | helm registry login --username AWS --password-stdin ${ACCOUNT_ID_EKS2}.dkr.ecr.eu-west-2.amazonaws.com
    - aws eks update-kubeconfig --region eu-west-2 --name pdu-${ENVIRONMENT}-eks-cluster
    - |
      set -eu
      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        helm upgrade --install --wait "${SERVICE_NAME}" "oci://${ACCOUNT_ID_EKS2}.dkr.ecr.eu-west-2.amazonaws.com/pdu-${ENVIRONMENT}/helm/generic-service" \
          --version "${GENERIC_SERVICE_HELM_CHART_VERSION}" \
          --namespace "${NAMESPACE}" \
          --history-max 2 \
          --values="values/pdu-${ENVIRONMENT}-values.yaml" \
          --set-string deployment.containers.main.image.tag="${SELECTED_IMAGE_TAG}" \
          --set-string deployment.containers.main.image.repository="${ECR_REPOSITORY}"
      else
        helm upgrade --install --wait "${SERVICE_NAME}" "oci://${ACCOUNT_ID_EKS2}.dkr.ecr.eu-west-2.amazonaws.com/pdu-${ENVIRONMENT}/helm/generic-service" \
          --version "${GENERIC_SERVICE_HELM_CHART_VERSION}" \
          --namespace "${NAMESPACE}" \
          --history-max 2 \
          --values="values/pdu-${ENVIRONMENT}-values.yaml"
      fi
    - |
      echo "Running post deployment status check..."
      kubectl config set-context --current --namespace=${NAMESPACE}
      if ! kubectl rollout status deployment/"${SERVICE_NAME}-main" --timeout=60s; then
        echo "Deployment failed for ${SERVICE_NAME}"
        exit 1
      fi

.kube-score: &kube-score
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/kube-score-helm-yq:1.1.0@sha256:3c293b09b0b57031d2a16247b44923150b25180b8aa0b1fc7dbf48df1b3b9fcf
  script:
    - VALUES_FILE="pdu-${ENVIRONMENT}-values.yaml"
    - helm repo add --username gitlab-ci-token --password $CI_JOB_TOKEN hcs-helm-charts ${CI_API_V4_URL}/projects/3913/packages/helm/stable
    - helm repo update
    - helm template "hcs-helm-charts/generic-service" --version=${GENERIC_SERVICE_HELM_CHART_VERSION} --values="values/${VALUES_FILE}" --debug | kube-score score -

.push-to-ecr: &push-to-ecr
  image: nexus.nonprod.dwpcloud.uk/docker-releases-dwp-secure-development/secure-development/container-image-factory/skopeo:1.2.0@sha256:9dc9d4507457616ea93b52fbf99681f51ab8817a1007640d9e3817cd44199563
  script:
    - aws ecr get-login-password --region eu-west-2 | skopeo login --username AWS --password-stdin ${ACCOUNT_ID_EKS2}.dkr.ecr.eu-west-2.amazonaws.com
    - |
      set -eu
      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        IMAGE_TAG=${SELECTED_IMAGE_TAG}
      else
        case "${ENVIRONMENT}" in
          dev) IMAGE_TAG=${DEV_IMAGE_TAG} ;;
          test) IMAGE_TAG=${TEST_IMAGE_TAG} ;;
          *) echo "Committed image selection is not configured for ${ENVIRONMENT}."; exit 1 ;;
        esac
      fi

      TARGET_IMAGE="${ECR_REPOSITORY}:${IMAGE_TAG}"
      SOURCE_IMAGE=
      SOURCE_DIGEST=

      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        SOURCE_REGISTRIES=${SELECTED_NEXUS_DOCKER_REGISTRY}
      else
        SOURCE_REGISTRIES=${NEXUS_RELEASE_DOCKER_REGISTRY}
      fi

      for SOURCE_REGISTRY in ${SOURCE_REGISTRIES}; do
        CANDIDATE_IMAGE="${SOURCE_REGISTRY}/${NEXUS_IMAGE_PATH}:${IMAGE_TAG}"
        set +e
        INSPECT_OUTPUT=$(skopeo inspect --format '{{.Digest}}' "docker://${CANDIDATE_IMAGE}" 2>&1)
        INSPECT_STATUS=$?
        set -e

        if [ "${INSPECT_STATUS}" -eq 0 ]; then
          CANDIDATE_DIGEST=$(printf '%s\n' "${INSPECT_OUTPUT}" | tail -n 1)
          if ! printf '%s\n' "${CANDIDATE_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
            echo "Nexus returned an invalid image digest for ${CANDIDATE_IMAGE}."
            exit 1
          fi
          SOURCE_IMAGE=${CANDIDATE_IMAGE}
          SOURCE_DIGEST=${CANDIDATE_DIGEST}
          break
        fi

        if ! printf '%s\n' "${INSPECT_OUTPUT}" | grep -Eqi 'manifest unknown|name unknown|manifest .*not found|manifest does not exist'; then
          printf '%s\n' "${INSPECT_OUTPUT}"
          exit "${INSPECT_STATUS}"
        fi
      done

      if [ -z "${SOURCE_IMAGE}" ]; then
        echo "Image ${IMAGE_TAG} does not exist in the available Nexus repositories."
        exit 1
      fi

      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        if [ -z "${BUILT_IMAGE_DIGEST:-}" ]; then
          echo "The built image digest is missing."
          exit 1
        fi
        if [ "${SOURCE_DIGEST}" != "${BUILT_IMAGE_DIGEST}" ]; then
          echo "The Nexus image digest does not match the image built by this pipeline."
          exit 1
        fi
      fi

      echo "Checking if image already exists in ECR: ${IMAGE_TAG}"
      set +e
      INSPECT_OUTPUT=$(skopeo inspect --format '{{.Digest}}' "docker://${TARGET_IMAGE}" 2>&1)
      INSPECT_STATUS=$?
      set -e

      if [ "${INSPECT_STATUS}" -eq 0 ]; then
        TARGET_DIGEST=$(printf '%s\n' "${INSPECT_OUTPUT}" | tail -n 1)
        if ! printf '%s\n' "${TARGET_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
          echo "ECR returned an invalid image digest for ${TARGET_IMAGE}."
          exit 1
        fi
        if [ "${SOURCE_DIGEST}" != "${TARGET_DIGEST}" ]; then
          echo "Existing ECR image does not match the source image digest."
          exit 1
        fi
        echo "Image ${IMAGE_TAG} already exists in ECR with digest ${TARGET_DIGEST} - skipping copy."
        {
          echo "IMAGE_COPY_VERIFIED=true"
          echo "VERIFIED_IMAGE_TAG=${IMAGE_TAG}"
          echo "VERIFIED_IMAGE_DIGEST=${TARGET_DIGEST}"
        } > image-copy.env
        exit 0
      fi

      if ! printf '%s\n' "${INSPECT_OUTPUT}" | grep -Eqi 'manifest unknown|name unknown|manifest .*not found|manifest does not exist'; then
        printf '%s\n' "${INSPECT_OUTPUT}"
        exit "${INSPECT_STATUS}"
      fi

      echo "Copying ${SOURCE_IMAGE} with digest ${SOURCE_DIGEST}"
      skopeo copy --override-arch=amd64 --override-os=linux "docker://${SOURCE_IMAGE}" "docker://${TARGET_IMAGE}"

      TARGET_DIGEST=$(skopeo inspect --format '{{.Digest}}' "docker://${TARGET_IMAGE}")
      if ! printf '%s\n' "${TARGET_DIGEST}" | grep -Eq '^sha256:[a-f0-9]{64}$'; then
        echo "ECR returned an invalid image digest for ${TARGET_IMAGE}."
        exit 1
      fi
      if [ "${SOURCE_DIGEST}" != "${TARGET_DIGEST}" ]; then
        echo "Source and target image digests do not match."
        exit 1
      fi
      {
        echo "IMAGE_COPY_VERIFIED=true"
        echo "VERIFIED_IMAGE_TAG=${IMAGE_TAG}"
        echo "VERIFIED_IMAGE_DIGEST=${TARGET_DIGEST}"
      } > image-copy.env
  artifacts:
    reports:
      dotenv: image-copy.env
    paths:
      - image-copy.env

.scan-ecr-vulns: &scan-ecr-vulns
  image: nexus.nonprod.dwpcloud.uk/docker/aws-cli/aws-cli:2.36.6@sha256:a9d9b72a496fd434172bef0a1cac0bf3812986184f9d3b3cbff65d6999e7c4f8
  script:
    - |
      set -eu
      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        IMAGE_TAG=${SELECTED_IMAGE_TAG}
      else
        case "${ENVIRONMENT}" in
          dev) IMAGE_TAG=${DEV_IMAGE_TAG} ;;
          test) IMAGE_TAG=${TEST_IMAGE_TAG} ;;
          *) echo "Committed image selection is not configured for ${ENVIRONMENT}."; exit 1 ;;
        esac
      fi

      if [ "${IMAGE_COPY_VERIFIED:-false}" != "true" ] || [ "${VERIFIED_IMAGE_TAG:-}" != "${IMAGE_TAG}" ]; then
        echo "Image copy verification is missing for ${IMAGE_TAG}."
        exit 1
      fi

      SCAN_STATUS=
      ATTEMPT=1
      while [ "${ATTEMPT}" -le 12 ]; do
        set +e
        SCAN_STATUS=$(aws ecr describe-image-scan-findings \
          --region eu-west-2 \
          --repository-name "${ECR_REPOSITORY_NAME}" \
          --image-id imageDigest="${VERIFIED_IMAGE_DIGEST}" \
          --query 'imageScanStatus.status' \
          --output text 2>/dev/null)
        SCAN_COMMAND_STATUS=$?
        set -e

        if [ "${SCAN_COMMAND_STATUS}" -eq 0 ] && [ "${SCAN_STATUS}" = "COMPLETE" ]; then
          break
        fi

        case "${SCAN_STATUS}" in
          FAILED|UNSUPPORTED_IMAGE|SCAN_ELIGIBILITY_EXPIRED|FINDINGS_UNAVAILABLE|LIMIT_EXCEEDED)
            echo "ECR scan failed with status ${SCAN_STATUS}."
            exit 1
            ;;
        esac

        if [ "${ATTEMPT}" -eq 12 ]; then
          echo "ECR scan results were not available for ${IMAGE_TAG}."
          exit 1
        fi

        echo "Waiting for ECR scan results for ${IMAGE_TAG}."
        sleep 10s
        ATTEMPT=$((ATTEMPT + 1))
      done

      CRITICAL_COUNT=$(aws ecr describe-image-scan-findings \
      --region eu-west-2 \
      --repository-name "${ECR_REPOSITORY_NAME}" \
      --image-id imageDigest="${VERIFIED_IMAGE_DIGEST}" \
      --query "length(imageScanFindings.findings[?severity=='CRITICAL'])" \
      --output text)
      echo "CRITICAL vulnerabilities found: ${CRITICAL_COUNT}"
      if [ "${CRITICAL_COUNT}" -gt 0 ]; then
        printf '%s CRITICAL vulnerabilities detected - cannot deploy this service\n' "$CRITICAL_COUNT"
        aws ecr describe-image-scan-findings \
          --region eu-west-2 \
          --repository-name "${ECR_REPOSITORY_NAME}" \
          --image-id imageDigest="${VERIFIED_IMAGE_DIGEST}" \
          --query "[
            imageScanFindings.findings[?severity=='CRITICAL'].[name,severity,uri],
            imageScanFindings.findings[?severity=='HIGH'].[name,severity,uri]
          ][]" \
          --output table
        exit 1
      fi

.docker-build-and-push-to-nexus: &docker-build-and-push-to-nexus
  script:
    - |
      set -eu
      if [ "${APP_CHANGED}" != "true" ]; then
        echo "No image build inputs changed - skipping build."
        echo "BUILT_IMAGE_DIGEST=" > built-image.env
        exit 0
      fi

      echo "Using HCS runners, commencing docker startup checks"
      for i in $(seq 1 30); do
        docker info && break
        echo "Waiting for docker to start"
        sleep 1s
      done

      NEXUS_USER=$(jq -r '.username' "${SELECTED_NEXUS_CREDENTIAL_FILE}")
      TOKEN=$(jq -r '.token' "${SELECTED_NEXUS_CREDENTIAL_FILE}")
      echo "${TOKEN}" | docker login -u "${NEXUS_USER}" --password-stdin "${SELECTED_NEXUS_DOCKER_REGISTRY}"

      cleanup() {
        STATUS=$?
        trap - EXIT INT TERM
        docker logout "${SELECTED_NEXUS_DOCKER_REGISTRY}" > /dev/null 2>&1 || true
        exit "${STATUS}"
      }
      trap cleanup EXIT
      trap 'exit 130' INT
      trap 'exit 143' TERM

      IMAGE="${SELECTED_NEXUS_DOCKER_REGISTRY}/${NEXUS_IMAGE_PATH}:${SELECTED_IMAGE_TAG}"
      set +e
      PULL_OUTPUT=$(docker pull "${IMAGE}" 2>&1)
      PULL_STATUS=$?
      set -e

      if [ "${PULL_STATUS}" -eq 0 ]; then
        EXISTING_BUILD_INPUTS=$(docker inspect --format '{{ index .Config.Labels "uk.gov.dwp.raco.build-inputs" }}' "${IMAGE}")
        if [ "${EXISTING_BUILD_INPUTS}" = "${BUILD_INPUTS_DIGEST}" ]; then
          BUILT_IMAGE_DIGEST=$(docker inspect --format '{{ index .RepoDigests 0 }}' "${IMAGE}" | sed 's/.*@//')
          echo "BUILT_IMAGE_DIGEST=${BUILT_IMAGE_DIGEST}" > built-image.env
          echo "Image ${SELECTED_IMAGE_TAG} already exists for these build inputs - reusing it."
          exit 0
        fi
        echo "Image ${SELECTED_IMAGE_TAG} already exists for different build inputs."
        exit 1
      fi

      if ! printf '%s\n' "${PULL_OUTPUT}" | grep -Eqi 'manifest unknown|manifest .*not found|manifest does not exist'; then
        printf '%s\n' "${PULL_OUTPUT}"
        exit "${PULL_STATUS}"
      fi

      docker build \
        --label "org.opencontainers.image.revision=${CI_COMMIT_SHA}" \
        --label "org.opencontainers.image.version=${SELECTED_IMAGE_TAG}" \
        --label "uk.gov.dwp.raco.build-inputs=${BUILD_INPUTS_DIGEST}" \
        -t "${IMAGE}" .
      docker push "${IMAGE}"
      docker pull "${IMAGE}"
      BUILT_IMAGE_DIGEST=$(docker inspect --format '{{ index .RepoDigests 0 }}' "${IMAGE}" | sed 's/.*@//')
      echo "BUILT_IMAGE_DIGEST=${BUILT_IMAGE_DIGEST}" > built-image.env
  artifacts:
    reports:
      dotenv: built-image.env
    paths:
      - built-image.env

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

      if [ "${DEPLOYMENT_PURPOSE:-committed}" = "application-pr" ]; then
        helm template "${SERVICE_NAME}" \
          hcs-helm-charts/generic-service \
          --version "${GENERIC_SERVICE_HELM_CHART_VERSION}" \
          --namespace "${NAMESPACE}" \
          --values "values/pdu-${ENVIRONMENT}-values.yaml" \
          --set-string deployment.containers.main.image.tag="${SELECTED_IMAGE_TAG}" \
          --set-string deployment.containers.main.image.repository="${ECR_REPOSITORY}" \
          > "msa-rendered/${ENVIRONMENT}/app.yaml"
      else
        helm template "${SERVICE_NAME}" \
          hcs-helm-charts/generic-service \
          --version "${GENERIC_SERVICE_HELM_CHART_VERSION}" \
          --namespace "${NAMESPACE}" \
          --values "values/pdu-${ENVIRONMENT}-values.yaml" \
          > "msa-rendered/${ENVIRONMENT}/app.yaml"
      fi

      test -s "msa-rendered/${ENVIRONMENT}/app.yaml"
  artifacts:
    paths:
      - msa-rendered/
    expire_in: 7 days

```
