### 파이프라인

- Rollout pipeline.yaml

  ```yaml
  apiVersion: tekton.dev/v1
  kind: Pipeline
  metadata:
    name: poc-app-rollout
  spec:
    params:
      - default: poc-app
        name: APP_NAME
        type: string
      - default: 'https://gitlab.apps.cluster-td4lk.td4lk.sandbox1543.opentlc.com/root/test.git'
        name: GIT_REPO
        type: string
      - default: main
        name: GIT_REVISION
        type: string
      - default: 'image-registry.openshift-image-registry.svc:5000/poc-app/poc-app'
        name: IMAGE_NAME
        type: string
      - default: .
        name: PATH_CONTEXT
        type: string
      - default: 20-ubi8
        name: VERSION
        type: string
    tasks:
      - name: fetch-repository
        params:
          - name: url
            value: $(params.GIT_REPO)
          - name: revision
            value: $(params.GIT_REVISION)
          - name: subdirectory
            value: ''
          - name: deleteExisting
            value: 'true'
        taskRef:
          kind: ClusterTask
          name: git-clone
        workspaces:
          - name: output
            workspace: workspace
      - name: build
        params:
          - name: IMAGE
            value: $(params.IMAGE_NAME)
          - name: TLS_VERIFY
            value: 'false'
          - name: CONTEXT
            value: $(params.PATH_CONTEXT)
          - name: VERSION
            value: $(params.VERSION)
        runAfter:
          - fetch-repository
        taskRef:
          kind: ClusterTask
          name: s2i-nodejs
        workspaces:
          - name: source
            workspace: workspace
      - name: rollout-deployment
        params:
          - name: SCRIPT
            value: |
              oc rollout restart deployment/$(params.APP_NAME) -n poc-app && \
              oc rollout status deployment/$(params.APP_NAME) -n poc-app
        runAfter:
          - build
        taskRef:
          kind: ClusterTask
          name: openshift-client
    workspaces:
      - name: workspace
  ```

- Event Listener Trigger 설정

  ```yaml
  apiVersion: triggers.tekton.dev/v1beta1
  kind: TriggerBinding
  metadata:
    name: poc-app
  spec:
    params:
      - name: git-repo-url
        value: $(body.repository.git_http_url)
      - name: git-repo-name
        value: $(body.repository.name)
      - name: git-revision
        value: $(body.ref)
  ---
  apiVersion: triggers.tekton.dev/v1beta1
  kind: TriggerTemplate
  metadata:
    name: poc-app
  spec:
    params:
      - description: The git repository url
        name: git-repo-url
      - default: pipelines-1.15.1
        description: The git revision
        name: git-revision
      - description: The name of the deployment to be created / patched
        name: git-repo-name
    resourcetemplates:
      - apiVersion: tekton.dev/v1
        kind: PipelineRun
        metadata:
          generateName: build-deploy-$(tt.params.git-repo-name)-
        spec:
          params:
            - name: APP_NAME
              value: poc-app
            - name: GIT_REPO
              value: 'https://gitlab.apps.cluster-td4lk.td4lk.sandbox1543.opentlc.com/root/test.git'
            - name: GIT_REVISION
              value: main
            - name: IMAGE_NAME
              value: 'image-registry.openshift-image-registry.svc:5000/poc-app/poc-app'
            - name: PATH_CONTEXT
              value: .
            - name: VERSION
              value: 20-ubi8
          pipelineRef:
            name: poc-app-rollout
          taskRunTemplate:
            serviceAccountName: pipeline
          timeouts:
            pipeline: 1h0m0s
          workspaces:
            - name: workspace
              volumeClaimTemplate:
                metadata:
                  creationTimestamp: null
                  labels:
                    tekton.dev/pipeline: poc-app
                spec:
                  accessModes:
                    - ReadWriteOnce
                  resources:
                    requests:
                      storage: 1Gi
  ---
  apiVersion: triggers.tekton.dev/v1beta1
  kind: Trigger
  metadata:
    name: poc-app-trigger
  spec:
    serviceAccountName: pipeline
    bindings:
      - ref: poc-app
    template:
      ref: poc-app
  ---
  apiVersion: triggers.tekton.dev/v1beta1
  kind: EventListener
  metadata:
    name: poc-app
  spec:
    serviceAccountName: pipeline
    triggers:
    - bindings:
      - ref: poc-app
      template:
        ref: poc-app
  ---
  apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    labels:
      app.kubernetes.io/managed-by: EventListener
      app.kubernetes.io/part-of: Triggers
      eventlistener: poc-app
    name: el-poc-app
  spec:
    port:
      targetPort: http-listener
    to:
      kind: Service
      name: el-poc-app
      weight: 100
  ```

  