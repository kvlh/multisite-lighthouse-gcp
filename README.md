# multisite-lighthouse-gcp
Run Lighthouse audits on URLs, and write the results daily into a BigQuery table.

# Steps 

1. Clone repo.
    `git clone https://github.com/kvlh/multisite-lighthouse-gcp`
2. Install [Google Cloud SDK](https://cloud.google.com/sdk/).
3. Authenticate with `gcloud auth login`.
4. Create a new GCP project.
5. Enable Cloud Functions API and BigQuery API.
6. Create a new dataset in BigQuery.
7. Run `gcloud config set project <projectId>` in command line.
    <projectId> in example config: multisite-lighthouse-gcp 
8. Edit `config.json`, 
  update list of `source` URLs and IDs, 
  edit `projectId` to your GCP project ID, 
  edit `pubsubTopicId` to the PubSub topic name.
  edit `datasetId` to the BigQuery dataset ID.
  edit `bucketName` to the bucket name.
    - create <bq dataset name> - in example config: lighthouse-bq 
    `bq mk <bq dataset name>`
    - <bucket name> in example config: lighthouse-reports 
    `gsutil mb -l eu -b on gs://<bucket name>`
    - create pubsub topic <pubsub topic> 
    `gcloud pubsub topics create <pubsub topic>`
    - deploy cloud function with enrtypoint launchLighthouse
    `gcloud functions deploy launchLighthouse --entry-point launchLighthouse --trigger-topic <pubsub topic> --memory 2048 --timeout 540 --runtime=nodejs16 --region=europe-central2`
9.  Run `gcloud pubsub topics publish launch-lighthouse --message all` to audit all URLs in source list.
10. Run `gcloud pubsub topics publish launch-lighthouse --message <source.id>` to audit just the URL with the given ID.
11. Verify with Cloud Functions logs and a BigQuery query that the performance data ended up in BQ. Might take some time, especially the first run when the BQ table needs to be created.



If you need to test mobile
1. Remove those 3 lines in config
    `"emulatedUserAgent": "constants.userAgents.desktop",
    "screenEmulation": {
      "disabled": true`
2. Change those 2 lines from desktop to mobile
   `    "preset": "desktop",
    "formFactor": "desktop",`


# How it works

When you deploy the Cloud Function to GCP, it waits for specific messages to be pushed into the `launch-lighthouse` Pub/Sub topic queue (this topic is automatically generated by the function).

When a message corresponding with a URL defined in `config.json` is registered, the function fires up a lighthouse instance and performs the basic audit on the URL.

This audit is then parsed into a BigQuery schema, and written into a BigQuery table named `report` under the dataset you created.

The BigQuery schema currently only includes items that have a "weight", i.e. those that impact the scores also provided in the audit. 

You can also send the message `all` to the Pub/Sub topic, in which case the Cloud Function self-executes a new function for every URL in the list, starting the lighthouse processes in parallel.

# Cost

This is extremely low-cost. You should basically be able to work with the free tier for a long while, assuming you don't fire the functions dozens of times per day. 

# UPDATE

Updated version of of Google cloud dependenies and lighthouse
Flat structure of BQ schema

