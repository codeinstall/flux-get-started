```
default:
  tags: [dwp-aws-x86-linux-l]
  retry:
    max: 2
    when:
      - runner_system_failure

include:
  - local: '/gitlab-ci/common-jobs.yaml'
  - local: '/gitlab-ci/stages.yaml'

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - when: never

smoke:aa-data-publisher:
  stage: smoke-test
  extends: .aa-data-publisher-smoke

smoke:landing-ui:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:journey-ui:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:decision-tree:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:orchestrator:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:pdf-generator:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:ecms-adapter:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:residency-adapter:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:audit-adapter:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:cryptology:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:token-manager:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

smoke:kong-gateway:
  stage: smoke-test
  extends: .disabled-smoke-test
  rules:
    - when: never

```
