# Multi-cluster configuration with Fleet Packages

[Fleet Packages] provides a way to deploy configuration into multiple clusters in a fleet.

## Prerequisistes

1. gcloud CLI, as this guide uses it to set up the necessary infrastructure and for 
2. A GitHub account, as we will fetch the configuration from git. Authentication to git will be set up using a GitHub App, so you must have admin-level permissions on your GitHub repository.
3. A Google Cloud project.

## Set up a fleet

The first step is to create a few GKE clusters and register them to Fleet. We only create two clusters here to keep things simple.

```
PROJECT=<my-project>
```

Create a cluster in us-west:
```
gcloud container clusters create fp-quickstart-nginx-west --project=$PROJECT --zone=us-west2-a --workload-pool=$PROJECT.svc.id.goog --enable-fleet --async
```

Create a cluster in us-east:
```
gcloud container clusters create fp-quickstart-nginx-east --project=$PROJECT --zone=us-east1-c --workload-pool=$PROJECT.svc.id.goog --enable-fleet --async
```

It will take a few minutes for the clusters to be ready. To verify that the clusters are ready, check that they are marked at `RUNNING` when running this command:

```
gcloud container clusters list --project=$PROJECT
```


## Install Config Sync across the fleet

```
gcloud beta container fleet config-management enable --fleet-default-member-config=apply-spec.yaml
```

## Configure Cloud Build Repositories

We have a sample package at https://github.com/GoogleCloudPlatform/anthos-config-management-samples/tree/main/fleet-packages-quickstart/config. This is a simple example that just sets up nginx. Fork this repo into your own GitHub account.

Fleet Packages leverages [Cloud Build Repositories] for connecting to git. Follow these steps to set it up:

1. Enable the Cloud Build and Secret Manager APIs:
```
gcloud services enable cloudbuild.googleapis.com
gcloud services enable secretmanager.googleapis.com
```

2. Verify the IAM permissions as described in https://cloud.google.com/build/docs/automating-builds/github/connect-repo-github?generation=2nd-gen#iam_perms.

3. Create a connection to your GitHub host using the steps described in https://cloud.google.com/build/docs/automating-builds/github/connect-repo-github?generation=2nd-gen#connecting_a_github_host. You can use either the console or gcloud. You must use the region `us-central1`.

4. Connect to the forked GitHub repository using the steps described in https://cloud.google.com/build/docs/automating-builds/github/connect-repo-github?generation=2nd-gen#connecting_a_github_repository_2. You can use either the console or gcloud. Again, you must use the region `us-central1`.

5. Make sure you remember the name of the connection and the repository you set up. If you are uncertain, you can list these with gcloud using the following commands:

List all connections:

```
gcloud builds repositories list --region=us-central1 --project=$PROJECT
```

List all repositories for a specific connection:

```
gcloud builds repositories list --region=us-central1 --connection=<CONNECTION_NAME>
```

Later in the guide we will need the fully qualified name for the connection. It will be:

```
projects/PROJECT/locations/us-central1/connections/CONNECTION_NAME/repositories/REPOSITORY_NAME
```

## Configure a Service Account

Create a service account:

```
gcloud iam service-accounts create fp-quickstart-sa
```

Add an IAM policy binding for the ResourceBundle Publisher role:

```
gcloud projects add-iam-policy-binding $PROJECT \
   --member="serviceAccount:fp-quickstart-sa@$PROJECT.iam.gserviceaccount.com" \
   --role='roles/configdelivery.resourceBundlePublisher'
```

Add an IAM policy binding for the [Logs Writer](https://cloud.google.com/iam/docs/understanding-roles#logging.logWriter) role:

```
gcloud projects add-iam-policy-binding PROJECT \
   --member="serviceAccount:fp-quickstart-sa@PROJECT.iam.gserviceaccount.com" \
   --role='roles/logging.logWriter'
```


## Create a FleetPackage

Create a file with the FleetPackage spec in yaml format:

```
cat << EOF > fp-spec.yaml
resourceBundleSelector:
  cloudBuildRepository:
    name: projects/$PROJECT/locations/us-central1/connections/CONNECTION_NAME/repositories/REPOSITORY_NAME
    tag: v1.0.0
    serviceAccount: projects/$PROJECT/serviceAccounts/fp-quickstart-sa@$PROJECT.iam.gserviceaccount.com
    path: config
target:
  fleet:
    project: projects/PROJECT
rolloutStrategy:
  rolling:
    maxConcurrent: 1
EOF
```

Create the FleetPackage:

```
gcloud alpha container fleet packages create fp-nginx \
    --source=fleetpackage-spec.yaml \
    --project=PROJECT
```
