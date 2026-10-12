# Deployment settings

Per-environment values for **connection references** and **environment variables**.

| File | Used when deploying to |
| --- | --- |
| `test.json` | Test |
| `prod.json` | Prod-Sim |

Both are optional: Deploy only passes a settings file to the import when it exists. Generate one with
`pac solution create-settings` — see [docs/setup.md](../docs/setup.md#5-connection-references-and-environment-variables-when-you-need-them).

Don't put secrets in these files. Use Azure Key Vault–backed environment variables for secret values.
