# Medical Claims Tools
<a href="https://github.com/google/medical_claims_tools/actions">
  <img src="https://github.com/google/medical_claims_tools/workflows/go_test/badge.svg" alt="GitHub Actions Build Status" />
</a>
<a href="https://godoc.org/github.com/google/medical_claims_tools">
  <img src="https://godoc.org/github.com/google/medical_claims_tools?status.svg" alt="Go Documentation" />
</a>

👀 _Please tell us more about your interest in or usage of these tools at our [survey here](https://docs.google.com/forms/d/e/1FAIpQLSdmWHaGc41gWiobMT6kNd0PGPPeWGeS-LyG6CrGZ79moaUIEQ/viewform)!_

This repository contains `bulk_fhir_fetch`, an ingestion tool that connects to [FHIR Bulk Data APIs](https://hl7.org/fhir/uv/bulkdata/) and saves the FHIR to local disk or GCP's
[FHIR Store](https://cloud.google.com/healthcare-api/docs/how-tos/fhir) and [BigQuery](https://cloud.google.com/bigquery). `bulk_fhir_fetch` is feature rich with support for scheduling and incremental data pulls, integrations to GCP logging/metrics, fetching binary data referenced by FHIR DocumentReferences, rectifying invalid FHIR and more. Popular FHIR Bulk Data APIs `bulk_fhir_fetch` can ingest data from include:

* Medicare's [Beneficiary Claims Data API (BCDA)](https://bcda.cms.gov/)
* The Cures Act [§170.315(g)(10) regulation](https://www.healthit.gov/test-method/standardized-api-patient-and-population-services) requires all US EHRs serve [USCDI](https://www.healthit.gov/isa/united-states-core-data-interoperability-uscdi#uscdi-v1) through a FHIR Bulk Data API

<br />

This is not an official Google product. __If using these tools with protected health information (PHI), please be sure
to follow your organization's policies with respect to PHI.__

## Overview
<!---TODO(b/199179306): add links to code paths below.--->
* `cmd/bulk_fhir_fetch/`: A program for fetching FHIR data from a
  FHIR Bulk Data API, and optionally saving to disk or sending to your
  FHIR Store. The tool
  is highly configurable via flags, and can support pulling incremental data
  only, among other features. See [bulk_fhir_fetch configuration examples](#bulk_fhir_fetch-configuration-examples) for details on how to use this program.
* `bulkfhir/`: A generic client package for interacting with FHIR Bulk Data APIs.
* `bcda/`: [Deprecated: prefer the generalized bulkfhir package] A go client package for interacting with the [BCDA](https://bcda.cms.gov/).
* `analytics/`: A folder with some analytics notebooks and examples.
* `fhirstore/`: A go helper package for uploading to FHIR store.
* `fhir/`: A go package with some helpful utilities for working with FHIR claims
  data.

## Set up bulk_fhir_fetch on GCP

The `bulk_fhir_fetch` command line program uses the `bulkfhir/` client library
to fetch FHIR data from a FHIR Bulk Data API. We documented __[how to set up a
GCP VM](docs/periodic_gcp_ingestion.md)__ to periodically ingest FHIR data using
`bulk_fhir_fetch`.

We are working on a second way to run `bulk_fhir_fetch` using GCP Scheduler, Workflow, Batch and Secret Manager. Work in progress scripts can be found in `orchestration/`.

## bulk_fhir_fetch Configuration Examples

This section will detail common usage patterns for the `bulk_fhir_fetch` command
line program using the [BCDA Sandbox](https://bcda.cms.gov/guide.html#try-the-api)
as an example. If you want to try this out __without__ using real credentials,
you can use the synthetic data sandbox credentials (client_id and client_secret)
from the options listed [here](https://bcda.cms.gov/guide.html#try-the-api). You
can check all of the various flag details by running `./bulk_fhir_fetch --help`.

__If using these tools with protected health information (PHI), please be sure
to follow your organization's policies with respect to PHI.__

* __Fetch all BCDA data for your ACO to local NDJSON files:__

  ```sh
  ./bulk_fhir_fetch \
    -client_id=YOUR_CLIENT_ID \
    -client_secret=YOUR_SECRET \
    -enable_generalized_bulk_import=true \
    -fhir_server_base_url="https://sandbox.bcda.cms.gov/api/v2" \
    -fhir_auth_url="https://sandbox.bcda.cms.gov/auth/token" \
    -output_dir="/path/to/store/output/data" \
  ```

* __Rectify the data to pass R4 Validation.__ By default, the FHIR R4 Data
returned by BCDA sandbox does not satisfy the default FHIR R4 profile at the time of
this software release. `bulk_fhir_fetch` provides an option to tag the expected missing
fields that BCDA does not map with an extension (if they are indeed missing)
that will allow the data to pass R4 profile validation (and be uploaded to FHIR
store, or other R4 FHIR servers). To do this, simply pass the following flag:

  ```sh
  -rectify=true
  ```

* __Fetch all claims data _since_ some timestamp__. This is useful if, for example,
you only wish to fetch new claims data since yesterday (or some other time).
Simply pass a [FHIR instant](https://www.hl7.org/fhir/datatypes.html#instant)
timestamp to the `-since` flag.

  ```sh
  -since="2021-12-09T11:00:00.123+00:00"
  ```
  Note that every time fetch is run, it will log the BCDA transaction time,
  which can be used in future runs of fetch to only get data since the last run.
  If you will be using fetch in this mode frequently, consider the since file
  option below which automates this behavior.

* __Automatically fetch new claims since last successful run.__ The program
provides a `-since_file` option, which the program uses to store and read BCDA
timestamps from successful runs. When using this option, the fetch program will
automatically read the latest timestamp from the since_file and use that to only
fetch claims since that time. When completed successfully, it will write a new
timestamp back out to that file, so that the next time fetch is run, only claims
since that time will be fetched. The first time the program is run with
`-since_file` it will fetch all historical claims from BCDA and initialize the
since_file with the first timestamp.

  ```sh
  -since_file="path/to/some/file"
  ```
Do not run concurrent instances of fetch that use the same since file.

* __Upload claims to a GCP FHIR Store:__

  ```sh
  ./bulk_fhir_fetch \
    -client_id=YOUR_CLIENT_ID \
    -client_secret=YOUR_SECRET \
    -enable_generalized_bulk_import=true \
    -fhir_server_base_url="https://sandbox.bcda.cms.gov/api/v2" \
    -fhir_auth_url="https://sandbox.bcda.cms.gov/auth/token" \
    -output_dir="/path/to/store/output/data/" \
    -rectify=true \
    -enable_fhir_store=true \
    -fhir_store_gcp_project="your_project" \
    -fhir_store_gcp_location="us-east4" \
    -fhir_store_gcp_dataset_id="your_gcp_dataset_id" \
    -fhir_store_id="your_fhir_store_id"
  ```

  Note: If `-enable_fhir_store=true` specifying `-output_dir` is optional. If
  `-output_dir` is not specified, no NDJSON output will be written to local
  disk and the only output will be to FHIR store. If you are using an older
  version of the tool, use `-output_prefix` instead of `-output_dir`.

To set up the `bulk_fhir_fetch` program to run periodically on a GCP VM, take a look at the
[documentation](docs/periodic_gcp_ingestion.md).

## Cloning at a pinned version

If cloning the repo for production use, we recommend cloning the repository at
the latest released version, which can be found in the
[releases](https://github.com/google/medical_claims_tools/releases)
tab. For example for version `v0.1.5`:

```sh
git clone --branch v0.1.5 https://github.com/google/medical_claims_tools.git
```

## Build

To build the program from source run the following from the root of the
repository (note you must have [Go](https://go.dev/dl/) installed):

```sh
go build cmd/bulk_fhir_fetch/bulk_fhir_fetch.go
```

This will build the `bulk_fhir_fetch` binary and write it out in your current
directory.

## Example Analytics
This repository also contains example [analysis notebooks](analytics)
using synthetic data that showcase query patterns once the data is in FHIR Store
and BigQuery.
