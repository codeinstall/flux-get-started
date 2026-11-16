```
import { expect, test } from '@playwright/test';

test.describe('AA data publisher', () => {
  test('reports healthy', async ({ request }) => {
    const response = await request.get('/health');

    expect(response.status()).toBe(200);
    expect(response.headers()['content-type']).toContain('text/plain');
    expect(await response.text()).toBe('Healthy');
  });

  test('reports live', async ({ request }) => {
    const response = await request.get('/live');

    expect(response.status()).toBe(200);
    expect(response.headers()['content-type']).toContain('text/plain');
    expect(await response.text()).toBe('Live');
  });
});

```
