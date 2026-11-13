```
kubernetesAPI:
  enabled: true
fullnameOverride: 'aa-data-publisher'
mandatoryLabels:
  application: 'report-a-change-once'
  function: 'digital-channels'
  environment: 'Dev'
aws:
  services:
    iam:
      enabled: true
      role: arn:aws:iam::943009210227:role/pdu-dev-dmae-raco-data-aa-aa-data-publisher

eventsPlatform:
  enabled: true

deployment:
  containers:
    main:
      env:
        NODE_ENV: development
        LOG_LEVEL: debug
        KAFKA_CLIENT_ID: raco-services
        KAFKA_PDF_GENERATOR_TOPIC: uk.gov.dwp.report_a_change_once.pdf_generator.2
        KAFKA_CHANGE_ADDRESS_TOPIC: uk.gov.dwp.report_a_change_once.change_of_res_address.2
        KAFKA_CONSUMER_GROUP_ID: uk.gov.dwp.report_a_change_once_aa_dev
        KAFKA_CONSUMER_FROM_BEGINNING: false
        KAFKA_BROKERS: dev-intra1-proxy.integr-events-platform-dev.dwpcloud.uk:9192
        SCHEMA_REGISTRY_URL: https://event-platform-waf.integr-events-platform-dev.dwpcloud.uk:8082/intra1
        TOKEN_MANAGER_URL: http://token-manager.dmae-raco-shared-token-manager.svc.cluster.local:8082/token
        KAFKA_CA_CERT: /mnt/secrets-store/events-platform-client-cert/ca.crt
        KAFKA_TLS_KEY: /mnt/secrets-store/events-platform-client-cert/tls.key
        KAFKA_TLS_CERT: /mnt/secrets-store/events-platform-client-cert/tls.crt
        KAFKA_PRODUCER_SCHEMA_VERSION: 1
      image:
        pullPolicy: Always
        repository: 943009210227.dkr.ecr.eu-west-2.amazonaws.com/pdu-dev/tenant/dmae-raco-data/aa-data-publisher
        tag: v1.0.0
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
          ephemeral-storage: 64M
        limits:
          cpu: 250m
          memory: 256Mi
          ephemeral-storage: 128M

      securityContext:
        runAsUser: 10001
        runAsGroup: 30001
        capabilities:
          drop: ['ALL']
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        seccompProfile:
          type: RuntimeDefault
      readinessProbe:
        httpGet:
          path: /health
          port: 8085
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 5
        successThreshold: 1
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /live
          port: 8085
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 5
        successThreshold: 1
        failureThreshold: 3
  mountDwpRootCa:
    enabled: true
  PodDisruptionBudget:
    maxUnavailable: 1

horizontalPodAutoscaler:
  enabled: true
  minReplicas: 2
  maxReplicas: 4
  additionalMetrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

service:
  name: aa-data-publisher
  ports:
    http:
      protocol: TCP
      port: 8085
      targetPort: 8085
      type: ClusterIP

certificates:
  events-platform-client-cert:
    domainName: 'uk.gov.dwp.report_a_change_once'
    duration: '168h0m0s'
    mountPath: '/mnt/secrets-store/events-platform-client-cert'

networkPolicy:
  egress:
    - namespace: dmae-raco-shared-token-manager
      app: token-manager
      port: 8082
    - namespace: dmae-raco-shared-pdf-generator
      app: pdf-generator
      port: 8083

```
