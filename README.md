```
{
  "name": "raco-shared-automation",
  "version": "1.0.0",
  "private": true,
  "description": "Shared automation tests for RACO services",
  "scripts": {
    "mock:aa-data-publisher": "node tools/mock-aa-data-publisher/server.mjs",
    "test": "npm run test:smoke",
    "test:smoke": "playwright test tests/smoke",
    "test:aa-data-publisher": "playwright test tests/smoke/aa-data-publisher",
    "test:integration": "playwright test tests/integration",
    "test:integration:aa-data-publisher": "playwright test tests/integration/aa-data-publisher",
    "test:report": "playwright show-report",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@kafkajs/confluent-schema-registry": "4.1.0",
    "@playwright/test": "1.56.1",
    "@types/node": "24.10.0",
    "kafkajs": "2.2.4",
    "typescript": "5.9.3"
  }
}

```
