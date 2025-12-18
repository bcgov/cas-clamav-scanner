# CAS ClamAV Malware Scanner

This repository is forked from the Google Cloud Malware Scanner repository, adapted to work for our CAS projects. 

## Terraform deployment to GCP for Malware Scanning

Forked from [the GCP Docker ClamAV malware scanner repo](https://github.com/GoogleCloudPlatform/docker-clamav-malware-scanner/tree/v3.5.0), which is used by [the GCP malware scanning solution documentation](https://cloud.google.com/solutions/automating-malware-scanning-for-documents-uploaded-to-cloud-storage).

For our use case, we use some changes to the terraform files to use GCS buckets for the terraform state. This ensures the state is not stored on an individual cloud console, but rather shared GCS. It's currently best to run the deployment from the GCP Cloud Shell to avoid extra confusion with credentials management.

### Infrastucture description

We used some slightly modified terraform files and configs, but the basics are still the same. This will deploy the following infrastructure:

- A set of service accounts for the malware scanner.
  - The accounts allow for the malware scanner to move files between buckets.
- An artifact registry repository for the malware scanner image.
- A Cloud Run service for the malware scanner (ClamAV).
- Unscanned Cloud Storage buckets.
- Clean Cloud Storage buckets.
- Quarantined Cloud Storage buckets.
- ClamAV malware definitions mirror bucket.
  - Cloud Scheduler job to update the ClamAV malware definitions mirror.
- Eventarc triggers on the unscanned buckets.

### Directions

#### Preparation

1. Log into GCP, and navigate to the cloud project you want to use for the deployment.
1. Enable requried APIs for the project.
    - Artifact Registry
    - Cloud Build
    - Cloud Run
    - Cloud Scheduler
    - Eventarc
    - Logging
    - Monitoring
    - Pub/Sub
    - Service Usage
    - Cloud Resource Manager
1. If you haven't already, create a storage bucket for the terraform state. Note, BCIERS `dev`/`test`/`prod` projects have a bucket already created for this purpose.
1. Enable required APIs for the project. Enable the Artifact Registry, Cloud Build, Resource Manager, Cloud Scheduler, Eventarc, Logging, Monitoring, Pub/Sub, Cloud Run, and Service Usage APIs. If you're already logged in to the proper project, [this link will enable the proper APIs](https://console.cloud.google.com/flows/enableapi?apiid=artifactregistry.googleapis.com,cloudbuild.googleapis.com,cloudresourcemanager.googleapis.com,cloudscheduler.googleapis.com,eventarc.googleapis.com,logging.googleapis.com,monitoring.googleapis.com,pubsub.googleapis.com,run.googleapis.com,serviceusage.googleapis.com)
1. Activate the Google Cloud Shell.
1. The rest of the directions for this deployment will be in the cloud shell.
1. Clone this repository into your shell (`git clone https://github.com/bcgov/cas-clamav-scanner.git`).
1. Navigate to the directory where you cloned the project.
1. Run `chmod 775 ./cloudrun-malware-scanner/updateCvdMirror.sh ./cloudrun-malware-scanner/bootstrap.sh` to ensure the mirror update script executable. This is run as part of the Terraform resource `null_resource.populate_cvd_mirror`.
1. In Cloud Shell, set common shell variables including region and location. Our projects use `northamerica-northeast1` (Montreal) for our storage regions/locations:

    ```bash
    REGION=northamerica-northeast1
    LOCATION=northamerica-northeast1
    PROJECT_ID=<the project id in GCP>
    OPENSHIFT_NAMESPACE=<the openshift project namespace, e.g. a1b2c3-dev>
    BUCKET_ROOT=<the base name for the bucket>
    STATE_BUCKET=<the name of the bucket to store the terraform state in>
    BCIERS_SERVICE_ACCOUNT=<the base name for the BCIERS service account, see ! below>
    export BUCKET_ROOT STATE_BUCKET
    ```

    > **!** The `BCIERS_SERVICE_ACCOUNT` variable is the base name of the service account created by the `terraform-bucket-provision` Helm chart. This is used to grant the service accounts access to the buckets. The pattern for these accounts is `"${openshift_namespace}-${app_name}"`. See the IAM & Admin > Service Accounts section of the GCP cloud console to find this value. *note*: This applies to the BCIERS project, other CAS projects might have multiple projects, and therefore multiple service accounts.
    >
    > From `BUCKET_ROOT` variable, 4 buckets will be created:
    > - `${BUCKET_ROOT}-unscanned`
    > - `${BUCKET_ROOT}-clean`
    > - `${BUCKET_ROOT}-quarantined`
    > - `${BUCKET_ROOT}-cvd-mirror`

1. Initialize the gcloud CLI environment with your project ID: `gcloud config set project "${PROJECT_ID}"`.
1. Configure the Terraform variables. This assumes you are in the in the `{repo}` directory. The contents of the config.json configuration file are passed to Terraform by using the TF_VAR_config_json variable, so that Terraform knows which Cloud Storage buckets are to create. The value of this variable is also passed to Cloud Run to configure the service.

```bash
TF_VAR_project_id=$PROJECT_ID
TF_VAR_region=$REGION
TF_VAR_bucket_location=$LOCATION
TF_VAR_config_json="$(envsubst < config/config.json)"
TF_VAR_create_buckets=true
TF_VAR_openshift_namespace=$OPENSHIFT_NAMESPACE
TF_VAR_bciers_service_account=$BCIERS_SERVICE_ACCOUNT
export TF_VAR_project_id TF_VAR_region TF_VAR_bucket_location TF_VAR_config_json TF_VAR_create_buckets TF_VAR_openshift_namespace TF_VAR_bciers_service_account
```

#### Deployment

##### Base infrastructure

1. Still within the same Cloud Shell, run the following commands. This assumes you are starting in the in the `{repo}` directory.

    ```bash
    gcloud services enable \
      cloudresourcemanager.googleapis.com \
      serviceusage.googleapis.com
    cd terraform/infra
    terraform init -backend-config="bucket=${STATE_BUCKET}"
    terraform apply
    ```

    **Double check any changes you see in the console output.** Respond with `yes` if everything looks good.

    > This Terraform script performs the following tasks:
    > - Creates the service accounts
    > - Creates the Artifact Registry
    > - Creates the Cloud Storage buckets
    > - Sets the appropriate roles and permissions
    > - Performs an initial population of the Cloud Storage bucket that contains the mirror of ClamAV malware definitions database

1. Build the container image for the malware scanner service. In the Cloud Shell, run the following commands to launch a Cloud Build job to create the container image for the service:

    ```bash
    cd ../../cloudrun-malware-scanner
    gcloud builds submit --region=$TF_VAR_region --config=cloudbuild.yaml \
        --service-account=projects/$PROJECT_ID/serviceAccounts/ms-build-$OPENSHIFT_NAMESPACE@$PROJECT_ID.iam.gserviceaccount.com \
        .
    ```

    Wait for the build to complete.

##### Service and triggers

1. In the Cloud Shell, run the following commands to deploy the Cloud Run service:

    ```bash
    cd ../terraform/service/
    terraform init -backend-config="bucket=${STATE_BUCKET}"
    terraform apply
    ```

    **Double check any changes you see in the console output.** Respond with `yes` if everything looks good.

    > It can take several minutes for the service to deploy and start. This terraform script performs the following tasks:
    > - Deploys the Cloud Run service by using the container image that you just built.
    > - Sets up the Eventarc triggers on the unscanned Cloud Storage buckets. Although your trigger is created immediately, it can take up to two minutes for that trigger to be fully functional.
    > - Creates the Cloud Scheduler job to update to the ClamAV malware definitions mirror.

    There is a chance that the deployment will fail with the following errors. If so, wait one minute and run `terraform apply` again.

    - `Error: Error creating Trigger: googleapi: Error 400: Invalid resource state for "": The request was invalid: Bucket "unscanned-ggl-cas-storage-dev" was not found. Please verify that the bucket exists.`
    - `Error: Error creating Trigger: googleapi: Error 400: Invalid resource state for "": Permission denied while using the Eventarc Service Agent. If you recently started to use Eventarc, it may take a few minutes before all necessary permissions are propagated to the Service Agent. Otherwise, verify that it has Eventarc Service Agent role..`

1. Verify that the service is running with the following command:

    ```bash
    MALWARE_SCANNER_URL="$(terraform output -raw cloud_run_uri)"
    curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" "${MALWARE_SCANNER_URL}"
    ```

    This command will output lines similar to:

    ```text
    gcs-malware-scanner version 3.2.0
    Using Clam AV version: ClamAV 1.4.1/27479/Fri Dec  6 09:40:14 2024
    ```

### Updating the cloudrun-malware-scanner image

If the upstream repository has updates in the `cloudrun-malware-scanner` directory, you can sync the changes from the upstream repo to this repo. Ensure you are on the latest main release, though depending on the major-minor-patch changes made, you may need to adapt them for this repo. You will then need to:

1. Follow the [preparation](#preparation) directions above to reinitialize your Cloud Shell. If your shell already has this repository cloned, you will need to run `git pull` to get the latest changes from `main`.
1. You may not need to run the all of the [Base Infrastructure deployment](#base-infrastructure) steps (depending on the changes made upstream), but you will need to run the container image build step. This will be run in the `{repo}/cloudrun-malware-scanner` directory.
1. Run the steps to [deploy the Cloud Run service](#service-and-triggers) again. This should use the latest built image.

### Testing and troubleshooting

#### Test file scanning

You can follow the directions in the [Google CAC documents](https://cloud.google.com/architecture/automate-malware-scanning-for-documents-uploaded-to-cloud-storage/deployment#test_the_pipeline_by_uploading_files).

#### Troubleshooting

##### EventArc `permission_denied`

If the *EventArc* just has `permission_denied`, check the *pub/sub* topic that's connected to it. There's a chance that certain APIs haven't been enabled. Click on the topic, then click *+ Trigger Cloud Run Function*. It should check that the required APIs are enabled. You can enable these automatically with the link in the [tutorial docs](https://cloud.google.com/architecture/automate-malware-scanning-for-documents-uploaded-to-cloud-storage/deployment#before_you_begin).

See [https://stackoverflow.com/questions/74313620/how-to-give-google-cloud-eventarc-correct-permission-so-it-can-trigger-a-cloud-f](https://stackoverflow.com/questions/74313620/how-to-give-google-cloud-eventarc-correct-permission-so-it-can-trigger-a-cloud-f), for further reference.

You might also get the following error in the same trigger sidebar.

```text
Service account(s) might not have enough permissions to deploy the function with selected trigger. To mitigate potential errors, you can grant them the following roles:

for service-############@gcp-sa-pubsub.iam.gserviceaccount.com:

roles/iam.serviceAccountTokenCreator on project-id
```

---
Original README from source repo:
---

# Malware Scanner Service

This repository contains the code to build a pipeline that scans objects
uploaded to GCS for malware, moving the documents to a clean or quarantined
bucket depending on the malware scan status.

It illustrates how to use Cloud Run and Eventarc to build such a pipeline.

![Architecture diagram](architecture.svg)

## How to use this example

Use the
[tutorial](https://cloud.google.com/solutions/automating-malware-scanning-for-documents-uploaded-to-cloud-storage)
to understand how to configure your Google Cloud Platform project to use Cloud
Run and Eventarc.

## Using Environment variables in the configuration

The tutorial above uses a configuration file `config.json` built into the Docker
container for the configuration of the unscanned, clean, quarantined and CVD
updater cloud storage buckets.

Environment variables can be used to vary the deployment in 2 ways:

### Expansion of environment variables

Any environment variables specified using shell-format within the `config.json`
file will be expanded using
[`envsubst`](https://manpages.debian.org/bookworm/gettext-base/envsubst.1.en.html).

### Passing entire configuration as environment variable

An alternative to building the configuration file into the container is to use
environmental variables to contain the configuration of the service, so that
multiple deployments can use the same container, and configuration updates do
not need a container rebuild.

This can be done by setting the environmental variable `CONFIG_JSON` containing
the JSON configuration, which will override any config in the `config.json`
file.

If using the `gcloud run deploy` command line, this environment variable must be
set using the
[`--env-vars-file`](https://cloud.google.com/sdk/gcloud/reference/run/deploy#--env-vars-file)
argument, specifying a YAML file containing the environment variable definitions
(This is because the commas in JSON would break the parsing of `--set-env-vars`)

Take care when embedding JSON in YAML - it is recommended to use the
[Literal Block Scalar style](https://yaml-multiline.info/) using `|`, as this
preserves newlines and quotes

For example, the `CONFIG_JSON` environment variable could be defined in a file
`config-env.yaml` as follows:

```yaml
CONFIG_JSON: |
  {
    "buckets": [
      {
        "unscanned": "unscanned-bucket-name",
        "clean": "clean-bucket-name",
        "quarantined": "quarantined-bucket-name"
      }
    ],
    "ClamCvdMirrorBucket": "cvd-mirror-bucket-name",
    "fileExclusionPatterns": [],
    ignoreZeroLengthFiles: false
  }
```

An example commandline using this file to specify the environment:

```sh
gcloud beta run deploy "${SERVICE_NAME}" \
  --source . \
  --region "${REGION}" \
  --no-allow-unauthenticated \
  --memory 4Gi \
  --cpu 1 \
  --concurrency 20 \
  --min-instances 1 \
  --max-instances 5 \
  --no-cpu-throttling \
  --cpu-boost \
  --service-account="${SERVICE_ACCOUNT}" \
  --env-vars-file=config-env.yaml
```

If you are using Terraform to deploy, then the equivalent way to specify the
environment variable using the
[`google_cloud_run_v2_service`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_v2_service)
resource is by using the
[env](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_v2_service#env)
block and
[jsonencode](https://developer.hashicorp.com/terraform/language/functions/jsonencode):

```tf
resource "google_cloud_run_v2_service" "malware-scanner" {
  name = "malware-scanner"
  // other service parameters...
  template {
    // other template parameters...
    containers {
      // other container parameters...
      env {
        name = "CONFIG_JSON"
        value = jsonencode({
          buckets = [
            {
              unscanned   = "unscanned-bucket-name",
              clean       = "clean-bucket-name",
              quarantined = "quarantined-bucket-name"
            }
          ]
          ClamCvdMirrorBucket = "cvd-mirror-bucket-name",
          fileExclusionPatterns = [],
          ignoreZeroLengthFiles = false
        })
      }
    }
  }
}
```

## Notes on `fileExclusionPatterns`

The `fileExclusionPatterns` array in the config file can be used to ignore any
uploaded files matching a
[Regular Expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions).

This can be used for example if you have an upload system that creates temporary
files, then renames them once the files are fully uploaded.

The elements in the `fileExclusionPatterns` array can either be simple strings,
for example:

```json
"fileExclusionPatterns": [
  "\\.tmp$",
  "^ignore_me.*\\.txt$"
]
```

or they can be an array of 2 string values, allowing regular expression flags to
be specified, for example `"i"` for case-insensitive matches:

```json
"fileExclusionPatterns": [
  [ "\\.tmp$", "i" ],
  [ "tempfile.*.upload$", "i" ]
]
```

Files matching these patterns will be ignored by the scanner, and left in the
`unscanned` bucket, and an `ignored-files` counter incremented.

Helpful tools for regular expressions include the
[Regular Expression Cheatsheet](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions/Cheatsheet),
and the [Regex101](https://regex101.com/r/QK47Hp/1) playground (ensure
ECMAScript flavor is selected).

Note that when adding regular expressions into the config file, care must be
taken with `\` and `"` characters -- any of these characters in the regular
expression must be escaped with another `\`.

## Notes on `quarantine` config block

The `quarantine` block in the config configures auto-quarantine of files which
do not have any malware detected, depending on certain factors:

### `quarantine.encryptedFiles`

(Default: `true`)

This enables ClamAV's encrypted file detection
([`AlertEncrypted` setting](https://manpages.debian.org/testing/clamav-daemon/clamd.conf.5.en.html#AlertEncrypted)),
which treats encrypted archive or docuemnt files as containing malware. This is
enabled by default as it is impossible to scan the contents of encyrpted files
for malware.

Encrypted files will be quarantined as if they were malware with a log line
indicating that they were infected with `Heuristics.Encrypted.xxx`.

### `quarantine.fileExtensionAllowList`

(Default: `[]` -- allow all.)

A list of allowed file extensions. Files not having these extensions will be
quarantined as if they were malware with a log line indicating that they were
infected with `Config.AllowList.Blocked`.

If no extensions are specified all files are allowed.

File extensions are case-insensitive. An empty string: `""` matches files with
no extension. Specifying double-extensions (eg `"tar.gz"` is supported).

Example: `[ "doc", "pdf", "jpg" ]`

### `quarantine.fileExtensionDenyList`

(Default: `[]` -- deny nothing.)

A list of denied file extensions. Files having these extensions will be
quarantined as if they were malware with a log line indicating that they were
infected with `Config.DenyList.Blocked`.

If no extensions are specified all files are allowed.

File extensions are case-insensitive. An empty string: `""` matches files with
no extension. Specifying double-extensions (eg `"tar.gz"` is supported).

Example: `[ "zip", "tar.gz", "jpg" ]`

## Change history

See [CHANGELOG.md](cloudrun-malware-scanner/CHANGELOG.md)

## Upgrading from v2.x to v3.x

In Version 3.x, the metrics reporting was changed to OpenTelemetry which uses a
different naming convention for metrics, so the metric names have changed from:

```text
custom.googleapis.com/opencensus/malware-scanning/METRIC-NAME
```

to

```text
workload.googleapis.com/googlecloudplatform/gcs-malware-scanning/METRIC-NAME
```

Any dashboards or alerts using these metrics must be updated

## Upgrading from v1.x to v2.x

Version 2 has a different way of handling ClamAV updates to avoid issues with
the ClamAV content distribution network.

See [upgrade_from_v1.md](upgrade_from_v1.md) for upgrading instructions.

## License

```text
Copyright 2022 Google LLC

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
```
