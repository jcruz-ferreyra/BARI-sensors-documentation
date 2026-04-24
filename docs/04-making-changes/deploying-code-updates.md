# Deploying Code Updates

This page documents the current manual deployment workflow for Azure Function Apps. At the moment, code is deployed from Visual Studio Code using the Azure Functions extension.

A future version of this project may use GitHub-based deployment or CI/CD once the project repository is hosted in the organization GitHub account. Until then, this page focuses only on the current deployment method and the network access requirements that apply during deployment.

## Current Deployment Method

Function App code is currently deployed manually from Visual Studio Code using the Azure extension.

This applies to the following Function App resources:

- `sensordata-func-sensor-rec-...`
- `sensordata-func-db-writer-...`
- `sensordata-func-api-...`

Each Function App should be deployed independently. Before deploying, confirm that the correct local project folder is open in VS Code and that the target Function App resource matches the code being deployed.

## Network Requirement for Deployment

Some Function Apps have public network access disabled during normal operation. This improves security, but it also means deployment from a local computer may fail unless public access is temporarily enabled.

Before deploying from VS Code, temporarily enable public access for the Function App being updated. After deployment is complete, disable public access again if that Function App does not need to remain publicly reachable.

## Which Function Apps Need Public Access?

| Function App | Normal Public Access | Deployment Note |
|---|---|---|
| `sensordata-func-sensor-rec-...` | Enabled | Must remain public because Particle Cloud sends webhook requests to it |
| `sensordata-func-db-writer-...` | Disabled | Temporarily enable public access before deploying from VS Code |
| `sensordata-func-api-...` | Enabled | Must remain public because the dashboard calls it |

The Database Writer is the main Function App where this matters. It normally runs internally and does not need to expose a public endpoint. However, local deployment tools still need to reach the Function App resource during deployment.

## Temporary Public Access Procedure

1. Open the Azure Portal.
2. Navigate to the Function App being deployed.
3. Open the networking settings.
4. Temporarily enable public network access.
5. Deploy the code from VS Code.
6. Confirm the deployment completed successfully.
7. Disable public network access again if the Function App should remain private.

!!! warning "Remember to Disable Public Access"
    If public access was enabled only for deployment, disable it immediately after deployment is complete. Leaving unnecessary public access enabled increases the system's exposure.

## Planned GitHub Deployment

GitHub-based deployment is not currently documented because the project repository and organization access are still pending.

Once GitHub deployment is configured, this page should be updated to explain:

- Which repository triggers deployments
- Which branch is used for production
- Whether deployments run through GitHub Actions
- Whether public network access is still required during deployment
- How to verify a successful deployment
- How to roll back a failed deployment
