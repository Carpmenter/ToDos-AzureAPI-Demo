# ASP.NET Core Web API Deployment to Azure

This repository contains an ASP.NET Core Web API project. This guide demonstrates how to set up continuous deployment to Azure Web Apps using GitHub Actions.

## Workflow Overview

The GitHub Actions workflow will build and deploy your .NET Core app to an Azure Web App when a commit is pushed to the `main` branch. This setup assumes that you have already created an Azure App Service web app. For instructions on setting up the Azure Web App, follow [this guide](https://docs.microsoft.com/en-us/azure/app-service/quickstart-dotnetcore?tabs=net60&pivots=development-environment-vscode).

## Workflow Configuration

1. **Download the Publish Profile:**
   - Go to the **Overview** page of your Web App in the Azure Portal.
   - Download the Publish Profile.
   - For more information, visit [this guide](https://docs.microsoft.com/en-us/azure/app-service/deploy-github-actions?tabs=applevel#generate-deployment-credentials).

2. **Create a GitHub Secret:**
   - Go to your GitHub repository.
   - Navigate to **Settings** > **Secrets and variables** > **Actions**.
   - Create a new secret named `AZURE_WEBAPP_PUBLISH_PROFILE` and paste the contents of the publish profile as its value.
   - For details on configuring the GitHub secret, check [this guide](https://docs.microsoft.com/azure/app-service/deploy-github-actions#configure-the-github-secret).

3. **Configure Workflow Variables:**
   - Update the `AZURE_WEBAPP_NAME` with the name of your Azure Web App.
   - Optionally, adjust `AZURE_WEBAPP_PACKAGE_PATH` and `DOTNET_VERSION` as needed.

## GitHub Actions Workflow

The workflow file is located at `azure-webapps-dotnet-core.yml` and includes the following steps:

```yaml
name: Build and deploy ASP.Net Core app to an Azure Web App

env:
  AZURE_WEBAPP_NAME: ToDo-API-NICA    # Set this to the name of your Azure Web App
  AZURE_WEBAPP_PACKAGE_PATH: '.'      # Set this to the path to your web app project, defaults to the repository root
  DOTNET_VERSION: '8.x'               # Set this to the .NET Core version to use

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up .NET Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Set up dependency caching for faster builds
        uses: actions/cache@v3
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/packages.lock.json') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Build with dotnet
        run: dotnet build --configuration Release

      - name: dotnet publish
        run: dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/myapp

      - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v3
        with:
          name: .net-app
          path: ${{env.DOTNET_ROOT}}/myapp

  deploy:
    permissions:
      contents: none
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: 'Development'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
      - name: Download artifact from build job
        uses: actions/download-artifact@v3
        with:
          name: .net-app
              
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Web App
        id: deploy-to-webapp
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
