# Qubit Capital Workflow Automation

This repository provides three main scripts designed to automate PDF generation and streamline sales call preparation for the Qubit Capital platform. The scripts are:

1. **PDF Generation Script (`PDFGenerationUsingDealID.py`)** - Automates the generation of PDF reports for sales calls by gathering data from various sources, including company websites and OpenAI's language model.
2. **tmux PDF Launcher (`CodeToLaunchTmux.py`)** - Utilizes tmux to launch detached sessions for generating PDFs based on deal IDs, allowing for concurrent processing of multiple deals.
3. **HubSpot Webhook Listener (`webhook.py`)** - A Flask-based server that listens for incoming webhook notifications from HubSpot, triggering PDF report generation for specified deal stages.

## PDF Generation Script (`PDFGenerationUsingDealID.py`)

This script generates comprehensive PDF reports for potential prospects by extracting information from various online sources.

### Dependencies

The following Python libraries are required:

- `requests`: For making HTTP requests.
- `pandas`: For data manipulation and analysis.
- `BeautifulSoup`: For web scraping.
- `reportlab`: For PDF generation.
- `pdfrw`: For PDF manipulation.
- `PyPDF2`: For reading PDF content.
- `openai`: For interacting with OpenAI's API.
- `tiktoken`: For token management with OpenAI's models.
- `requests_toolbelt`: For handling multipart form data.

#### Installation

```bash
pip install requests pandas beautifulsoup4 reportlab pdfrw PyPDF2 openai tiktoken requests-toolbelt
```

### Configuration

Set up a configuration file named `configFile.json` with the following keys:

```json
{
    "AZURE_OPENAI_API_KEY": "your_api_key",
    "AZURE_OPENAI_ENDPOINT": "your_endpoint",
    "azure_openai_api_version": "your_api_version",
    "azure_openai_deployment_name": "your_deployment_name",
    "gptModel": "your_model_name",
    "maxNumOfToken": 1000,
    "jinaAIApiKey": "your_jina_api_key",
    "rapidApiKey": "your_rapid_api_key",
    "googleAPIKey": "your_google_api_key",
    "googleSearchEngineID": "your_google_search_engine_id",
    "googleCustomSearchURL": "your_google_custom_search_url",
    "hubspotAccessToken": "your_hubspot_access_token"
}
```

### Key Functions

- **`mainFunc(dealID)`**: Entry point for generating the PDF report.
- **`toCrawlTargetCompanyWebsiteFunc(dealID, website, country)`**: Crawls the target company's website for data.
- **`googleQueriesFunc(dealID, targetWebsite, industry, country)`**: Performs Google searches for additional insights.
- **`generate_pdf(file_path, openAIResponse, appendix)`**: Generates the PDF report.
- **`uploadPDFonHubspotFunc(filePath, hubspotPDFFolderPath)`**: Uploads the PDF to HubSpot.
- **`isPDFAlreayGeneratedFunc(dealID)`**: Checks if a PDF report has already been generated.

### Usage

To run the script, execute the following command:

```bash
python3 PDFGenerationUsingDealID.py <dealID>
```

## tmux PDF Launcher (`CodeToLaunchTmux.py`)

This script launches new tmux sessions for generating PDFs based on deal IDs, facilitating concurrent processing of multiple deals.

### Functionality

- **`launchingTmuxForGeneratingPDF(dealID)`**: Launches a new tmux session for generating a PDF based on the provided deal ID.

#### Usage

To use this function:

1. Ensure tmux is installed.
2. Call the function with a valid deal ID:

```python
result = launchingTmuxForGeneratingPDF("DEAL001")
print(result)
```

### Benefits

- **Concurrent Processing**: Multiple PDF generations can run simultaneously.
- **Resource Management**: Detached sessions optimize system resources.
- **Error Handling**: Errors are isolated to specific sessions.

---

## HubSpot Webhook Listener (`.py`)

This Flask application listens for incoming webhook notifications from HubSpot, triggering PDF report generation for specific deal stages.

### Key Functions

- **`hubspot_webhook()`**: Handles incoming webhook requests and triggers PDF generation if the deal stage matches predefined IDs.

### Usage

To run the Flask application, execute:

```bash
python webhook.py
```

Replace `your_script_name.py` with the actual filename. The application will start a web server on `localhost:5001`.

### Example Usage

You can send a POST request to the `/hubspot_webhook` endpoint:

```bash
curl -X POST http://localhost:5001/hubspot_webhook \
-H "Content-Type: application/json" \
-d '[
    {"objectId": "deal_id_1", "propertyValue": "stage_id_1"}
]'
```

---

## Conclusion

These scripts together provide a comprehensive workflow for automating PDF generation and preparing sales teams at Qubit Capital for effective sales calls. Each script is designed for specific tasks, ensuring efficiency and scalability in handling multiple deals concurrently.
