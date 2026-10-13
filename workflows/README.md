# Exported workflows

This folder contains the three workflows of the project, exported from n8n as JSON.

| File | Workflow | Role |
|---|---|---|
| `01_fetch_dataset_sample.json` | Fetch Dataset Sample Subworkflow | Downloads a file from Kaggle, unzips it if needed, extracts it and samples its rows. Used by Cases 3 and 4. |
| `02_run_tabpfn.json` | Run TabPFN Subworkflow | Builds the training and test sets and calls the TabPFN-3 REST API. Used by Case 4. |
| `03_main_workflow.json` | Main Workflow | Chat Trigger, classification Agent 1, Switch and the four paths. |

## Exporting them from your instance

From the UI: open the workflow, top-right menu, **Download**.

From the command line, with the container running:

```bash
docker exec n8n n8n export:workflow --all --separate --output=/tmp/workflows
docker cp n8n:/tmp/workflows ./workflows-export
```

Do not export credentials with `export:credentials`: even though they are encrypted, they must not be published.

## Check before publishing

An exported workflow only contains a reference to its credentials, but a key typed by hand into an HTTP field or a system message would end up in the JSON. Before every commit:

```bash
grep -rniE "tabpfn_sk|bearer |api[_-]?key|AIza" workflows/
```

The command must return nothing. If a key shows up, remove it from the workflow and revoke it from the service's dashboard.
