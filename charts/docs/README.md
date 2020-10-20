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
                --set docs.ingress.enabled=true
                --set docs.ingress.annotations."kubernetes\.io/ingress\.class"="nginx"
                --set docs.ingress.annotations."nginx\.ingress\.kubernetes\.io/proxy-connect-timeout"="60s"
                --set docs.ingress.annotations."nginx\.ingress\.kubernetes\.io/proxy-send-timeout"="60s"
                --set docs.ingress.annotations."nginx\.ingress\.kubernetes\.io/proxy-read-timeout"="60s"
                --set docs.ingress.annotations."cert-manager\.io/issuer"="letsencrypt-prod"
                --set docs.ingress.tls[0].hosts[0]="docs-${{ parameters.namespace }}.${{ parameters.dns_tld }}"
                --set docs.ingress.tls[0].secretName="letsencrypt-tls-docs"
                --set docs.ingress.hosts[0].host="docs-${{ parameters.namespace }}.${{ parameters.dns_tld }}"
                --set docs.ingress.hosts[0].paths[0]="/"
            displayName: Deploy docs service
