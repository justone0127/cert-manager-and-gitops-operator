## OADP

### 1. httpd Template 작성 및 적용

- 설정 파일 위치 : https://github.com/justone0127/httpd-hello.git
  - httpd-template.yaml
  - index.html

- 테스트 시 사용할 httpd template로 `openshift` 네임스페이스 적용하여 사용합니다.

  ```yaml
  apiVersion: template.openshift.io/v1
  kind: Template
  metadata:
    name: httpd-poc-demo
    namespace: openshift
    annotations:
      iconClass: icon-apache
      openshift.io/display-name: HTTPD PoC App
      openshift.io/documentation-url: https://github.com/sclorg/httpd-ex
      openshift.io/provider-display-name: Red Hat, Inc.
      openshift.io/support-url: https://access.redhat.com
      samples.operator.openshift.io/version: 4.16.25
      tags: quickstart,httpd
    labels:
      samples.operator.openshift.io/managed: "true"
  parameters: []
  objects:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: httpd-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
  - apiVersion: v1
    kind: Service
    metadata:
      annotations:
        description: Exposes and load balances the application pods
      name: ${NAME}
    spec:
      ports:
      - name: web
        port: 8080
        targetPort: 8080
      selector:
        name: ${NAME}
  - apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: ${NAME}
    spec:
      host: ${APPLICATION_DOMAIN}
      to:
        kind: Service
        name: ${NAME}
  - apiVersion: image.openshift.io/v1
    kind: ImageStream
    metadata:
      annotations:
        description: Keeps track of changes in the application image
      name: ${NAME}
  - apiVersion: build.openshift.io/v1
    kind: BuildConfig
    metadata:
      annotations:
        description: Defines how to build the application
        template.alpha.openshift.io/wait-for-ready: "true"
      name: ${NAME}
    spec:
      output:
        to:
          kind: ImageStreamTag
          name: ${NAME}:latest
      source:
        contextDir: ${CONTEXT_DIR}
        git:
          ref: ${SOURCE_REPOSITORY_REF}
          uri: ${SOURCE_REPOSITORY_URL}
        type: Git
      strategy:
        sourceStrategy:
          from:
            kind: ImageStreamTag
            name: httpd:${HTTPD_VERSION}
            namespace: ${NAMESPACE}
        type: Source
      triggers:
      - type: ImageChange
      - type: ConfigChange
      - github:
          secret: ${GITHUB_WEBHOOK_SECRET}
        type: GitHub
      - generic:
          secret: ${GENERIC_WEBHOOK_SECRET}
        type: Generic
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      annotations:
        description: Defines how to deploy the application server
        image.openshift.io/triggers: |
          [{"from":{"kind":"ImageStreamTag","name":"${NAME}:latest"},"fieldPath":"spec.template.spec.containers[0].image"}]
        template.alpha.openshift.io/wait-for-ready: "true"
      name: ${NAME}
    spec:
      replicas: 1
      selector:
        matchLabels:
          name: ${NAME}
      strategy:
        type: RollingUpdate
      template:
        metadata:
          labels:
            name: ${NAME}
          name: ${NAME}
        spec:
          containers:
          - env: []
            image: ${NAME}:latest
            livenessProbe:
              httpGet:
                path: /
                port: 8080
              initialDelaySeconds: 30
              timeoutSeconds: 3
            name: httpd-poc-demo
            ports:
            - containerPort: 8080
            readinessProbe:
              httpGet:
                path: /
                port: 8080
            volumeMounts:
              - name: html-volume
                mountPath: /usr/local/apache2/htdocs # /opt/app-root/src 로 바꿔서 가능한지 테스트 부탁 드립니다. 
          volumes:
          - name: html-volume
            persistentVolumeClaim:
              claimName: httpd-pvc
  parameters:
  - description: The name assigned to all of the frontend objects defined in this template.
    displayName: Name
    name: NAME
    required: true
    value: httpd-poc-app
  - description: The OpenShift Namespace where the ImageStream resides.
    displayName: Namespace
    name: NAMESPACE
    required: true
    value: openshift
  - description: Version of HTTPD image to be used (2.4-el8 by default).
    displayName: HTTPD Version
    name: HTTPD_VERSION
    required: true
    value: 2.4-el8
  - description: Maximum amount of memory the container can use.
    displayName: Memory Limit
    name: MEMORY_LIMIT
    required: true
    value: 512Mi
  - description: The URL of the repository with your application source code.
    displayName: Git Repository URL
    name: SOURCE_REPOSITORY_URL
    required: true
    value: https://gitlab.apps.cluster-hfqtc.hfqtc.sandbox2220.opentlc.com/root/hello.git
  - description: Set this to a branch name, tag or other ref of your repository if you
      are not using the default branch.
    displayName: Git Reference
    name: SOURCE_REPOSITORY_REF
  - description: Set this to the relative path to your project if it is not in the root
      of your repository.
    displayName: Context Directory
    name: CONTEXT_DIR
  - description: The exposed hostname that will route to the httpd service, if left
      blank a value will be defaulted.
    displayName: Application Hostname
    name: APPLICATION_DOMAIN
  - description: Github trigger secret.  A difficult to guess string encoded as part
      of the webhook URL.  Not encrypted.
    displayName: GitHub Webhook Secret
    from: '[a-zA-Z0-9]{40}'
    generate: expression
    name: GITHUB_WEBHOOK_SECRET
  - description: A secret string used to configure the Generic webhook.
    displayName: Generic Webhook Secret
    from: '[a-zA-Z0-9]{40}'
    generate: expression
    name: GENERIC_WEBHOOK_SECRET
  ```

  > index.html을 PVC에 마운트 잡도록 설정하였습니다. 아래 파라미터 부분에서 자신의 git 주소로 대체하여 사용하시면 됩니다.

- index.html 내용

  ```html
  <!DOCTYPE html>
  <html lang="en">
  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Keco PoC Test Page</title>
      <style>
          body {
              background-color: #add8e6; /* Light blue background */
              font-family: 'Comic Sans MS', cursive, sans-serif;
              text-align: center;
              padding: 50px;
          }
  
          h1 {
              font-size: 3em;
              color: #0000ff; /* Blue color for the title */
              text-shadow: 2px 2px #87ceeb; /* Light blue shadow effect */
          }
  
          p {
              font-size: 1.5em;
              color: #4682b4; /* Steel blue color for the text */
              margin-top: 20px;
              font-family: 'Arial', sans-serif; /* Updated to a more elegant font for Korean */
          }
      </style>
  </head>
  <body>
      <h1>Keco PoC!!</h1>
      <p>안녕하세요!! 좋은 하루 되세요!!</p>
  </body>
  </html>
  ```

- 템플릿 적용

  ```bash
  oc apply -f httpd.yaml -n openshift
  ```

- 콘솔 확인

  **HTTPD PoC APP** 이름으로 확인할 수 있으며, 해당 템플릿을 이용하여 애플리케이션을 배포합니다.

  ![httpd_template](C:\Works\01_자료\01_OCP\2024_한국환경공단\images\httpd_template.png)

### 2. 애플리케이션 배포

- 프로젝트를 생성하여 위의 템플릿으로 애플리케이션을 배포합니다.

- pod 상태를 보면 detail에서 PVC가 마운트 되어 있는 것을 볼 수 있습니다. 

  ![httpd_volume](C:\Works\01_자료\01_OCP\2024_한국환경공단\images\httpd_volume.png)

- 애플리케이션 호출

  http 프로토콜로 호출합니다.

  ![httpd_index_pages](C:\Works\01_자료\01_OCP\2024_한국환경공단\images\httpd_index_pages.png)

### 3. OADP로 백업

- 백업 인스턴스 생성 시 **includedNamespaces** 지정 후 백업
- 프로젝트 삭제
- 복구 인스턴스 생성 시 **backupName** , **includedNamespaces**, **restorePVs** 체크 후 진행
- 페이지가 정상적으로 호출되는지와 Pod에 볼륨이 붙어 있는지 확인합니다.