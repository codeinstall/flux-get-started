```
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: Boolean(process.env['CI']),
  retries: process.env['CI'] ? 2 : 0,
  workers: process.env['CI'] ? 1 : undefined,
  reporter: process.env['CI']
    ? [
        ['list'],
        ['html', { open: 'never' }],
        ['junit', { outputFile: 'test-results/junit.xml' }],
      ]
    : 'list',
  use: {
    baseURL: process.env['AA_DATA_PUBLISHER_BASE_URL'] ?? 'http://localhost:8085',
    extraHTTPHeaders: {
      Accept: 'text/plain',
    },
    trace: 'retain-on-failure',
  },
});

```
