---
description: A complete example of all features in nais.yaml
---
# Max example

```
apiVersion: "nais.io/v1alpha1"
kind: "Application"
metadata:
  name: nais-testapp
  namespace: default
  labels:
    team: aura
spec:
  image: navikt/nais-testapp:65.0.0
  port: 8080
  strategy:
    type: RollingUpdate
  liveness:
    path: isalive
    port: http
    initialDelay: 20
    timeout: 1
    periodSeconds: 5
    failureThreshold: 10
  readiness:
    path: isready
    port: http
    initialDelay: 20
    timeout: 1
  replicas:
    min: 2
    max: 4
    cpuThresholdPercentage: 50
  prometheus:
    enabled: false
    path: /metrics
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 200m
      memory: 256Mi
  ingresses:
    - "https://nais-testapp.nais.adeo.no/"
    - "https://tjenester.nav.no/nais-testapp"
  vault:
    enabled: false
    sidecar: false
    paths:
      - mountPath: /var/run/secrets/nais.io/vault
        kvPath: /kv/preprod/fss/application/namespace
  configMaps:
    files:
      - example_files_configmap
  env:
    - name: MY_CUSTOM_VAR
      value: some_value
  preStopHookPath: "/stop"
  leaderElection: false
  webproxy: false
  logformat: accesslog
  logtransform: http_loglevel
  secureLogs:
    enabled: false
  service:
    port: 80
  secrets:
    - name: my-secret
      type: env
    - name: my-secret-file
      type: file
      mountPath: /var/run/secrets
  skipCaBundle: false
  accessPolicy:
    inbound:
      rules:
        - application: app1
        - application: app2
          namespace: q1
        - application: *
          namespace: t1
    outbound:
      rules:
        - application: app4
        - application: app1
          namespace: q1
        - application: *
          namespace: t1
      external:
        - host: www.external-application.com
 ```
