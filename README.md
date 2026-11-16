```
.disabled-smoke-test:
  script:
    - exit 1

.aa-data-publisher-smoke:
  image: nexus.nonprod.dwpcloud.uk/docker/dwp.gov.uk/node:26.12.1@sha256:e940ccfaf8e63b50558dd6bd1d830b07ba7ead78a7e599220e575d3c185dffd7
  script:
    - npm ci
    - |
      set -eu

      node tools/mock-aa-data-publisher/server.mjs > mock-aa-data-publisher.log 2>&1 &
      MOCK_PID=$!

      cleanup() {
        if kill -0 "${MOCK_PID}" 2>/dev/null; then
          kill "${MOCK_PID}"
          wait "${MOCK_PID}" || true
        fi
      }

      on_exit() {
        STATUS=$?
        trap - EXIT INT TERM
        cleanup
        exit "${STATUS}"
      }

      trap on_exit EXIT
      trap 'exit 130' INT
      trap 'exit 143' TERM

      MOCK_READY=false
      for attempt in $(seq 1 30); do
        if ! kill -0 "${MOCK_PID}" 2>/dev/null; then
          echo "AA Data Publisher mock stopped before becoming ready."
          cat mock-aa-data-publisher.log
          exit 1
        fi

        if node -e "fetch('http://127.0.0.1:8085/health', { signal: AbortSignal.timeout(1000) }).then(response => { if (!response.ok) process.exit(1) }).catch(() => process.exit(1))"; then
          MOCK_READY=true
          break
        fi
        sleep 1
      done

      if [ "${MOCK_READY}" != "true" ]; then
        echo "AA Data Publisher mock did not become ready after 30 attempts."
        cat mock-aa-data-publisher.log
        exit 1
      fi

      AA_DATA_PUBLISHER_BASE_URL=http://127.0.0.1:8085 npm run test:aa-data-publisher
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: test-results/junit.xml
    paths:
      - mock-aa-data-publisher.log
      - playwright-report/
      - test-results/
```
