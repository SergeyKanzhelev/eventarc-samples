# Eventarc Pub/Sub and Workflows Integration

In this sample, you will see how to connect
[Eventarc](https://cloud.google.com/eventarc/docs) events to
[Workflows](https://cloud.google.com/workflows/docs).

More specifically, you will see how a Pub/Sub message to a topic triggers a
Cloud Run service via Eventarc. In turn, the Cloud Run service executes a
workflow with whole HTTP request from Eventarc passed to the workflow.

## Deploy a workflow

First, deploy the [workflow.yaml](../eventarc-workflows-integration/eventarc-pubsub/workflow.yaml). It simply
decodes and logs out the received Pub/Sub message.

Deploy workflow:

```sh
export WORKFLOW_NAME=workflow-pubsub
export REGION=us-central1
gcloud workflows deploy ${WORKFLOW_NAME} --source=workflow.yaml --location=${REGION}
```

## Deploy a Cloud Run service to execute the workflow

Next, deploy a Cloud Run service to execute workflow. It simply executes the
workflow with the HTTP request. You can see the source code in
[trigger-workflow](../eventarc-workflows-integration/eventarc-pubsub/trigger-workflow).

Build the container:

```sh
export PROJECT_ID=$(gcloud config get-value project)
export SERVICE_NAME=trigger-workflow-pubsub
gcloud builds submit --tag gcr.io/${PROJECT_ID}/${SERVICE_NAME} .
```

Deploy the service:

```sh
gcloud config set run/region ${REGION}
gcloud config set run/platform managed
gcloud run deploy ${SERVICE_NAME} \
  --image gcr.io/${PROJECT_ID}/${SERVICE_NAME} \
  --region=${REGION} \
  --allow-unauthenticated \
  --update-env-vars GOOGLE_CLOUD_PROJECT=${PROJECT_ID},WORKFLOW_REGION=${REGION},WORKFLOW_NAME=${WORKFLOW_NAME}
```

## Connect a Pub/Sub topic to the Cloud Run service

Connect a Pub/Sub topic to the Cloud Run service by creating an Eventarc Pub/Sub
trigger:

```sh
gcloud config set eventarc/location ${REGION}
gcloud eventarc triggers create ${SERVICE_NAME} \
  --destination-run-service=${SERVICE_NAME} \
  --destination-run-region=${REGION} \
  --location=${REGION} \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished"
```

Find out the Pub/Sub topic that Eventarc created:

```sh
export TOPIC_ID=$(basename $(gcloud eventarc triggers describe ${SERVICE_NAME} --format='value(transport.pubsub.topic)'))
```

## Trigger the workflow

Send a message to the Pub/Sub topic to trigger the workflow:

```sh
gcloud pubsub topics publish ${TOPIC_ID} --message="Hello there"
```

In the logs for Workflows, you should see that Workflow received the Pub/Sub
message and decoded it.
