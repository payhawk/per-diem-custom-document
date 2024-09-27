This is an example of how to integrate seamlessly with Payhawk, adding custom functionality on top of what is already supported. In this example we showcase two integration points:
1. Subscribing and consuming a Payhawk webhook
2. Using the Payhawk API

This sample script solves the actual problem of generating a custom document for each per-diem expense in Payhawk. The code can be easily modified to work with other types of expenses or be executed on a different expense event.

In this case Google Docs and Google Cloud Platform are used to showcase how to host the custom functionality in the cloud. Google-specific APIs and functionalities are utilized. It is possible to use other tools and technologies, for example Microsoft Office 365 and Microsoft Azure. The Payhawk API is an HTTP API and as such is agnostic of how you call it. Note, that the hosting solution (e.g. Google Cloud Platform) is used in the background and Payhawk users do not need to have user accounts there or even know about it.

HOW IT WORKS

1. An expense is approved in Payhawk
2. Payhawk calls a cloud function deployed in Google Cloud Platform through a webhook
3. The cloud function uses the Payhawk API to get data related to the approved expense
4. The cloud function opens the template document and replaces the predefined placeholders with data from the expense
5. The cloud function exports the resulting document in PDF format
6. The cloud function uses the Payhawk API to attach the PDF to the expense
7. All Payhawk users with access to the expense can now see the PDF document from within the Payhawk portal


REQUIREMENTS

- A Payhawk account
- A Google Cloud Platform account (free tier will be enough)
- A Google Drive account
- Google CLI


HOW TO SET UP

Set up the template document:
1. Create a folder for the template file
2. Put the template in the folder (the ID of the template document will be needed later)

Set up the cloud function:
1. Create the cloud function in Google Cloud Platform
    - 2nd gen
    - Trigger: HTTPS
    - Variables
        - PAYHAWK_API_KEY -> the API key for your Payhawk account
        - GDOCS_TEMPLATE_FILE_ID -> the ID of the Google Docs file
        - PAYHAWK_ACCOUNT_ID -> the ID of your Payhawk account
        - WEBHOOK_EVENT_NAME -> Payhawk webhook event to subscribe to (e.g. "expense.approved")

Set up permissions:
1. Allow the cloud function to be invoked without authentication, so that it can be invoked by Payhawk
    > gcloud run services add-iam-policy-binding {cloud-function-name} --member="allUsers" --role="roles/run.invoker"

2. Set up a new service account
    1. Create a new service account in the GCP project

3. Set up the cloud function to run with the permissions of the service account
    1. Set the cloud function to use the new service account (Edit->Runtime)

4. Share the Google Drive folder that contains the template with the service account (use the service account email)

Enable Google APIs:
You need to enable some APIs in the GCP project that contains the cloud function
    - Enable Google Docs API (https://console.developers.google.com/apis/api/docs.googleapis.com/overview)
    - Enable Google Drive API (https://console.cloud.google.com/apis/library/drive.googleapis.com)

Deploy the cloud function:
1. Set the current project (only if not already set)
> gcloud config set project {gcp-project-nmame}
2. Deploy the code
> gcloud functions deploy {cloud-function-name} --region europe-west1 --gen2 --source=dist --set-build-env-vars=GOOGLE_NODE_RUN_SCRIPTS="" --entry-point=main --runtime=nodejs20 --trigger-http

Initialize the cloud function:
The following will call the cloud function to initialize itself. The initialization is needed to set up Payhawk to call the cloud function when an expense is approved.
> curl -X POST {cloud_function_url}?mode=init -H "Content-Type: application/json" -d "{\"webhookUrl\": \"{cloud_function_url}?mode=webhook"}"


HOW TO RUN LOCALLY
Running the cloud function locally:
> npm run dev
This will run a local server that accepts HTTP requests. You need to make na HHTP request to actually call the function.

To call the initialization:
> curl -X POST http://localhost:8080/?mode=init

To generate a document for an expense:
> curl -X POST http://localhost:8080/?mode=webhook -H "Content-Type: application/json" -d "{\"payload\": {\"expenseId\": \"{id-of-expense}\"}}"


