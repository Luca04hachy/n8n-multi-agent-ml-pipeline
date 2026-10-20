# n8n Multi-Agent ML Pipeline

A multi-agent [n8n](https://n8n.io) workflow that routes natural-language machine-learning requests to one of four inference strategies, without training any model locally: direct answer, Zero-Shot Learning, Few-Shot Learning with examples from Kaggle, and tabular In-Context Learning with TabPFN-3. The reasoning is done by Google Gemini (Gemini 3.6 Flash).

The project accompanies the bachelor's thesis *Studio e progettazione di un sistema multi-agente in n8n* (University of Modena and Reggio Emilia, UNIMORE).

## How it works

A first agent (`AI Agent - Prompt Analysis & Workflow Setup`) receives the request from the chat, searches Kaggle for a relevant dataset and picks one of four paths.

| Path | Triggered when | Strategy |
|---|---|---|
| Case 1 | the request cannot be fulfilled, or a concrete input value is missing | Direct answer to the user |
| Case 2 | no suitable dataset is found | Zero-Shot Learning (Basic LLM Chain) |
| Case 3 | the dataset is textual | Few-Shot Learning with examples sampled from the dataset |
| Case 4 | the dataset is tabular | TabPFN-3 through its REST API, with two agents in sequence (Setup and Predict) |

All paths converge on the `Output to Chat` node, which returns the final answer to the user.

> **Language.** The prompts and the replies in the current configuration are in Italian, because the system was developed and tested in Italian. Nothing in the architecture depends on it: to change the reply language, edit the system messages of the `AI Agent - Prompt Analysis & Workflow Setup` and `AI Agent - TabPFN Predict` nodes. The comments on the JavaScript scripts are also in Italian.

## Requirements

- Docker with Docker Compose v2
- A Kaggle account and its API key
- A Google Gemini API key (the project uses Gemini 3.6 Flash)
- A Prior Labs API key for TabPFN-3 ([https://platform.priorlabs.ai/](https://platform.priorlabs.ai/))

## Quick start

```bash
cp .env.example .env
# Generate a key with: openssl rand -hex 32
# then paste it after N8N_ENCRYPTION_KEY= in .env
docker compose up -d

# Check that the container is running
docker compose ps

# Show n8n's logs in real time (Ctrl+C exits the logs, but n8n keeps running)
docker compose logs -f n8n
```

> `N8N_ENCRYPTION_KEY` is required: Docker Compose refuses to start if it is empty. n8n uses it to encrypt the credentials stored in its database, so keep a copy in a safe place and never commit your `.env` file.

n8n is then available at http://localhost:5678. On first launch, create the owner account.

To stop n8n without removing the containers: `docker compose stop` (resume with `docker compose start`).

To stop and remove the containers: `docker compose down`. Your data stays in the `n8n_data` volume, so `docker compose up -d` brings everything back. `docker compose down -v` also deletes the volume, and with it all your workflows and credentials.

Tested with n8n 2.25.7 and Gemini 3.6 Flash.

## Importing the workflows

The order matters, because the main workflow calls the other two as tools. n8n does not let you publish a workflow until every credential used by its nodes has been created and selected, so set up the credentials as you go (see the table below).

1. Import `workflows/01_fetch_dataset_sample.json`, select your Kaggle credential in its nodes, then **publish** it.
2. Import `workflows/02_run_tabpfn.json`, select your Prior Labs credential and, in the node that calls `Fetch Dataset Sample`, the workflow imported in step 1. Then publish it.
3. Import `workflows/03_main_pipeline.json`, select your Kaggle and Gemini credentials and, in the tool nodes `Fetch_Dataset_Sample` and `Run_TabPFN`, the two imported workflows. Workflow IDs change at every import, so these links must be re-created.

> The two sub-workflows must be **published**: if they are not, the call from the main workflow is not executed. The main workflow only needs to be published if you want to use it outside the n8n editor.

## Credentials to create in n8n

| Service | Credential type | Used by |
|---|---|---|
| Kaggle | Basic Auth (username and API key) | `Search_Dataset`, `List_Kaggle_Dataset_Files`, `Download_Dataset_File` |
| Prior Labs | Header Auth: name `Authorization`, value `Bearer <your key>` | HTTP Request nodes of `Run_TabPFN` |
| Google Gemini | native n8n credential | Google Gemini Chat Model node |

No credentials are included in this repository.

## Example prompts

The prompts below are the ones used for testing, in Italian, with an English translation.

- Case 1: "Quanti abitanti avrà la prima colonia umana su Marte nel 2150?" (*How many inhabitants will the first human colony on Mars have in 2150?*)
- Case 2: "Ho una piccola pasticceria di quartiere: per la settimana di Ferragosto conviene puntare di più su cornetti alla crema o al cioccolato?" (*I run a small neighbourhood pastry shop: for the week of Ferragosto, should I focus more on custard or chocolate croissants?*)
- Case 3: "L'email 'Hai vinto 1000 euro! Clicca sul link per riscuotere!' è spam?" (*Is the email "You won 1000 euros! Click the link to collect!" spam?*)
- Case 4, classification: "Voglio sapere se un uomo ha il diabete: pesa 102 kg, glicemia superiore a 200 mg/dL, HbA1c superiore al 9%, e 66 anni di età." (*I want to know whether a man has diabetes: 102 kg, blood glucose above 200 mg/dL, HbA1c above 9%, aged 66.*)
- Case 4, regression: "Quanto costeranno in un anno le spese mediche di una persona di 45 anni, fumatrice, con BMI 33 e due figli?" (*How much will the yearly medical costs of a 45-year-old smoker with a BMI of 33 and two children be?*)

The dataset is selected through the Kaggle search query that the agent composes, so outcomes may vary between runs.

## Known limitations

- TabPFN predictions can vary across identical runs, because the training sample is re-drawn at every call.
- `AI Agent - Prompt Analysis & Workflow Setup`'s conversational memory lasts for the whole chat session: unrelated requests in the same session can influence each other.
- Only classification and regression tasks on text requests are supported.

## License

MIT, see `LICENSE`. TabPFN-3 and Gemini are third-party services with their own terms of use.
