# Setup

About 30 minutes, once. You need:

- Three Power Platform **developer environments** (Dev, Test, Prod-Sim) — see
  [how to set them up for free](https://joeysmith.dev/blog/free-power-platform-dev-environment/)
- An account that is a **Global Administrator** or **Power Platform Administrator** in that tenant
- The [Power Platform CLI](https://learn.microsoft.com/en-us/power-platform/developer/cli/introduction) (`pac`),
  e.g. via the **Power Platform Tools** VS Code extension

## 1. Create a solution in Dev

1. In [make.powerapps.com](https://make.powerapps.com), switch to your **Dev** environment.
2. **Solutions → New solution.** Give it a display name and a **Name** without spaces, e.g. `ALMDemo`.
   Pick (or create) a publisher with a short prefix, e.g. `js`.
3. Add something to it: a table, a canvas or model-driven app, a cloud flow. Anything you build
   inside the solution is what this pipeline moves between environments.

The solution's **Name** (not display name) is what you'll put in `SOLUTION_NAME`.

## 2. Create the service principal

The workflows sign in as an app (service principal), not as you. One command creates the app in
Microsoft Entra ID and adds it to Dev as an application user with the System Administrator role:

```bash
pac auth create --environment https://<dev-org>.crm.dynamics.com
pac admin create-service-principal --environment https://<dev-org>.crm.dynamics.com
```

It prints the **Tenant ID**, **Application ID** and a **Client Secret**. Copy all three now — the secret
is only shown once.

Add the same app to Test and Prod-Sim:

```bash
pac admin assign-user --environment https://<test-org>.crm.dynamics.com --user <application-id> --role "System Administrator" --application-user
pac admin assign-user --environment https://<prod-org>.crm.dynamics.com --user <application-id> --role "System Administrator" --application-user
```

> Find each environment's URL in the [Power Platform admin center](https://admin.powerplatform.microsoft.com)
> → Manage → Environments → (environment) → Environment URL.

## 3. Configure the GitHub repo

**Settings → Secrets and variables → Actions**

| Kind | Name | Value |
| --- | --- | --- |
| Secret | `PP_APP_ID` | Application ID from step 2 |
| Secret | `PP_CLIENT_SECRET` | Client secret from step 2 |
| Secret | `PP_TENANT_ID` | Tenant ID from step 2 |
| Variable | `SOLUTION_NAME` | Solution name from step 1, e.g. `ALMDemo` |
| Variable | `DEV_ENVIRONMENT_URL` | `https://<dev-org>.crm.dynamics.com` |

**Settings → Environments**

Create two environments, each with a variable named `ENVIRONMENT_URL`:

| Environment | `ENVIRONMENT_URL` | Protection |
| --- | --- | --- |
| `test` | Test environment URL | none |
| `production` | Prod-Sim environment URL | **Required reviewers:** add yourself |

The required reviewer is the approval gate: deployments wait for you before touching Prod-Sim.

**Settings → Actions → General → Workflow permissions**

Turn on **Allow GitHub Actions to create and approve pull requests** (the export workflow opens PRs).

## 4. First run

1. **Actions → Export from Dev → Run workflow.** It exports the solution, unpacks it into
   `solutions/<name>/`, opens a pull request, and starts **Validate** on that branch.
2. Review the PR. Wait for **Validate** (pack + Solution Checker) to pass.
3. **Merge.** **Deploy** builds the managed package, imports it into **Test**, then waits for your
   approval before importing into **Prod-Sim**.

## 5. Connection references and environment variables (when you need them)

If the solution contains flows with connections, or environment variables, each target environment
needs its own values. Generate a template from a packed solution:

```bash
pac solution pack --zipfile out/ALMDemo_managed.zip --folder solutions/ALMDemo --packagetype Managed
pac solution create-settings --solution-zip out/ALMDemo_managed.zip --settings-file deployment-settings/test.json
```

Fill in the connection IDs and values for Test, copy it to `prod.json` for Prod-Sim, and commit.
Deploy picks up `deployment-settings/<env>.json` automatically when the file exists.

> Connections must already exist in the target environment, owned by (or shared with) the service
> principal. Connection IDs are environment-specific; they aren't secrets.

## Troubleshooting

- **`Export from Dev` can't open the PR** — check the workflow-permissions setting in step 3.
- **Import fails with a missing dependency** — something in the solution depends on a component that
  isn't in the solution. Add it to the solution in Dev (Add existing → …) and export again.
- **Import fails on a flow** — the flow's connection reference has no connection in the target
  environment; see step 5.
- **Service principal gets "user is not a member of the organization"** — it wasn't added to that
  environment; rerun the `assign-user` command for it.
