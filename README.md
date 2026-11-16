```
import { randomUUID } from 'node:crypto';
import { expect, test } from '@playwright/test';
import {
  createKafka,
  createRegistry,
  EventProducer,
  SchemaName,
  type NonAuditEvent,
} from '@dwp-raco/dwp-event-components';
import type { Consumer } from 'kafkajs';

const requiredEnvironment = (name: string): string => {
  const value = process.env[name];
  if (!value) {
    throw new Error(`${name} must be set for the data publisher integration test.`);
  }
  return value;
};

const waitWithTimeout = <T>(promise: Promise<T>, timeout: number, description: string): Promise<T> => {
  let timer: NodeJS.Timeout;
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout(() => reject(new Error(`Timed out waiting for ${description}.`)), timeout);
  });

  return Promise.race([promise, timeoutPromise]).finally(() => clearTimeout(timer));
};

test.describe('AA data publisher Kafka integration', () => {
  test('publishes an address change event for AA', async () => {
    const brokers = requiredEnvironment('AA_DATA_PUBLISHER_KAFKA_BROKERS').split(',');
    const registryUrl = requiredEnvironment('AA_DATA_PUBLISHER_SCHEMA_REGISTRY_URL');
    const inputTopic = requiredEnvironment('AA_DATA_PUBLISHER_INPUT_TOPIC');
    const outputTopic = requiredEnvironment('AA_DATA_PUBLISHER_OUTPUT_TOPIC');
    const schemaVersion = Number(requiredEnvironment('AA_DATA_PUBLISHER_SCHEMA_VERSION'));
    const timeout = Number(process.env['AA_DATA_PUBLISHER_TEST_TIMEOUT_MS'] ?? '30000');

    requiredEnvironment('KAFKA_CA_CERT');
    requiredEnvironment('KAFKA_TLS_KEY');
    requiredEnvironment('KAFKA_TLS_CERT');
    requiredEnvironment('TOKEN_MANAGER_URL');

    if (!Number.isInteger(schemaVersion) || schemaVersion < 1) {
      throw new Error('AA_DATA_PUBLISHER_SCHEMA_VERSION must be a positive integer.');
    }
    if (!Number.isFinite(timeout) || timeout < 1) {
      throw new Error('AA_DATA_PUBLISHER_TEST_TIMEOUT_MS must be a positive number.');
    }
    test.setTimeout(timeout + 15000);

    const correlationId = randomUUID();
    const event: NonAuditEvent = {
      head: {
        headerVersion: '1.1.0',
        bodyTemplateType: 'ThinEvent',
        bodyTemplateVersion: '1.1.0',
        eventUuid: randomUUID(),
        eventName: 'address-change-automation-test',
        eventDateTime: new Date().toISOString(),
        publishedDateTime: new Date().toISOString(),
        correlationId,
        journeyInstanceId: randomUUID(),
        subjectId: randomUUID(),
        subjectType: 'DWP_GUID',
      },
      body: {
        benefits: ['AA'],
        benefit: null,
        journeyStartedDateTime: new Date().toISOString(),
        citizenDetails: {
          nino: 'QQ123456C',
          forename: 'Automation',
          surname: 'Test',
          dateOfBirth: '01-01-1970',
        },
        answers: [],
        metadata: [{ field: 'testType', value: 'data-publisher-integration' }],
        documentUploadData: null,
        errorDetails: null,
      },
    };

    const kafka = createKafka({
      clientId: `raco-shared-automation-${correlationId}`,
      brokers: brokers.map((broker) => broker.trim()),
    });
    const registry = await createRegistry(registryUrl);
    const producer = new EventProducer(kafka, registry, SchemaName.NON_AUDIT, {
      schemaIdentifier: {
        subject: `${inputTopic}-value`,
        version: schemaVersion,
      },
    });
    const consumer: Consumer = kafka.consumer({
      groupId: `raco-shared-automation-${correlationId}`,
    });

    let resolveReceived = (_received: NonAuditEvent): void => {};
    const received = new Promise<NonAuditEvent>((resolve) => {
      resolveReceived = resolve;
    });
    const consumerReady = new Promise<void>((resolve) => {
      consumer.on(consumer.events.GROUP_JOIN, () => resolve());
    });

    try {
      await consumer.connect();
      await consumer.subscribe({ topic: outputTopic, fromBeginning: false });
      await consumer.run({
        eachMessage: async ({ message }) => {
          if (!message.value) return;
          const candidate = (await registry.decode(message.value)) as NonAuditEvent;
          if (candidate.head.correlationId === correlationId) {
            resolveReceived(candidate);
          }
        },
      });
      await waitWithTimeout(consumerReady, timeout, 'the output consumer to join its group');

      await producer.connect();
      await producer.publishEvent(event, inputTopic);

      const publishedEvent = await waitWithTimeout(
        received,
        timeout,
        `event ${correlationId} on ${outputTopic}`
      );

      expect(publishedEvent.head.correlationId).toBe(correlationId);
      expect(publishedEvent.body).toEqual({
        ...event.body,
        benefit: 'AA',
      });
    } finally {
      await producer.disconnect().catch(() => undefined);
      await consumer.stop().catch(() => undefined);
      await consumer.disconnect().catch(() => undefined);
    }
  });
});

```
