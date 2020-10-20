# Docs chart

This chart is to host a documentation container, well basically it can be any simple container...


Sample for using with Azure DevOps:

```
variables:
- name: project_name
  value: your-project
- name: docs_help_repo_name
  value: valtechdocs
- name: docs_help_repo_url
  value: https://valtechdk.github.io/helm-chart-docs
- name: docs_chart_name
  value: helm-chart-docs/docs

- stage: deploy_testing
  displayName: Deploy (testing)
  dependsOn: build
  pool:
    vmImage: ubuntu-latest
  jobs:
  - deployment: deploy
    displayName: Deploy
    environment: '${{ parameters.environment_name }}.${{ parameters.namespace }}'
    strategy:
        runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              connectionType: Kubernetes Service Connection
              kubernetesServiceEndpoint: ${{ parameters.k8s_admin_connection }}
              namespace: ${{ parameters.namespace }}
              command: repo
              arguments: add $(docs_help_repo_name) $(docs_help_repo_url)
            displayName: Add docs Helm repository

          - task: HelmDeploy@0
            inputs:
              connectionType: Kubernetes Service Connection
              kubernetesServiceEndpoint: ${{ parameters.k8s_admin_connection }}
              namespace: ${{ parameters.namespace }}
              command: repo
              arguments: >-
                update
            displayName: Update Helm repository

          - task: HelmDeploy@0
            inputs:
              connectionType: Kubernetes Service Connection
              kubernetesServiceEndpoint: ${{ parameters.k8s_admin_connection }}
              namespace: ${{ parameters.namespace }}
              command: upgrade
              chartType: Name
              chartName: $(docs_chart_name)
              releaseName: ${{ parameters.project_name }}-docs
              install: true
              waitForExecution: true
              arguments: >-
                --timeout 30m0s
                --values $(Pipeline.Workspace)/Windows/docs-config.yaml
                --set tolerations[0].key="os"
                --set tolerations[0].operator="Equal"
                --set tolerations[0].value="windows"
                --set tolerations[0].effect="NoSchedule"
                --set ingress.enabled=true
                --set ingress.hosts[0].host="docs-${{ parameters.namespace }}.${{ parameters.dns_tld }}"
                --set ingress.hosts[0].paths[0]="/"
            displayName: Deploy docs service
