# Updating the Dashboard

## Overview

Explain that the dashboard is a static web app deployed from GitHub. Changes are made in the repository and automatically deployed after pushing to the configured branch.

## What Can Be Changed

Include the main safe change categories:

- Text labels and UI wording
- Styling in `css/styles.css`
- API base URL or config values in `js/config.js`
- Chart behavior in `js/charts.js`
- Dashboard behavior in `js/dashboard.js`
- Heat/noise page layout in `heat.html` or `noise.html`

## Before Making Changes

This is important:

- Confirm whether the change affects heat page, noise page, or both.
- Confirm whether the change is frontend-only or requires API changes.
- Avoid changing the `deployment_id` URL parameter unless all embedded links are also updated.
- Do not add secrets to frontend files.

## Deployment Workflow

Short:

- Edit files in the GitHub repository.
- Commit and push to the deployment branch.
- GitHub deployment updates the Azure Static Web App automatically.
- Check the deployment status in GitHub or Azure Static Web App.

## Post-Deployment Checks

This is the most useful section:

- Open heat page with a known `deployment_id`.
- Open noise page with a known `deployment_id`.
- Confirm date range selector works.
- Confirm aggregation buttons work.
- Confirm metric tabs work on heat page.
- Confirm comparison dropdown loads.
- Confirm API calls return 200 in browser Network tab.
- Confirm no visible chart errors.

## Common Changes

Examples:

- Change dashboard title or labels.
- Change default date range.
- Change max selectable date range.
- Add or rename a metric tab.
- Update API base URL.
- Change chart colors or labels.
- Change comparison behavior.

## Things to Be Careful With

Important:

- `deployment_id` is legacy but required for existing links.
- Frontend metric names must match API-supported metric names.
- API response format must match what `charts.js` expects.
- Static frontend files are public.
- CDN dependencies affect page loading.

## Related Documentation

Reference static web app, API endpoints, function code.
