# 🧊 Snowstorm-Local Setup Guide

This document is a step-by-step guide to launching the containerized **Snowstorm terminology server**, customized by **E-health**. It includes default behavior, customization tips, and verification steps to ensure everything runs smoothly.

📎 **GitHub Repository**: [https://github.com/ehealthplatformstandards/snowstorm](https://github.com/ehealthplatformstandards/snowstorm)

Snowstorm supports multiple terminologies. For more details:
- [**Supported Terminologies**](syndication/syndication-terminologies.md)
- [**Startup Configuration**](syndication/syndication-on-startup.md)
- [**Runtime Updates**](syndication/syndication-on-runtime.md)

---

## 🚀 Quick Launch Overview

Using the default `docker-compose.yml`, the following terminologies are **imported** at runtime startup:

- **ATC**, **BCP13**, **BCP47**, **ISO3166**, **ISO3166-2**, **UCUM** *(embedded in Docker image)*

Additionally, the following are **downloaded and imported**:
- The latest versions of **LOINC**, **HL7**, and **SNOMED CT** (including the **Belgian extension** and **international edition**)

You can customize what to load via Docker launch arguments. ElasticSearch must be running and properly configured before starting Snowstorm.

🕒 **Note**: Initial setup may take up to **40 minutes**, depending on internet speed and hardware.

---

## ⚙️ Step 1: (Optional) Create Accounts

To access **SNOMED** and **LOINC**, you must register if you don't already have an account:
- SNOMED: [https://mlds.ihtsdotools.org/#/register](https://mlds.ihtsdotools.org/#/register) *(requires approval after account creation)*
- LOINC: [https://loinc.org/join/](https://loinc.org/join/)

---

## ⚙️ Step 2: Configure `.env`

Edit your `.env` file with the following:

- Set a **secure value** for `SYNDICATION_SECRET` to enable runtime updates.
- Add credentials for **SNOMED CT** and **LOINC** download if using those terminologies.

---

## ⚙️ Step 3 (Optional): Customize Launch Options

The `command` section of `docker-compose.yml` controls the behavior of Snowstorm at startup. Example:

```yaml
command: [
  "--elasticsearch.urls=http://es:9200",
  "--syndication",
  "--atc=https://raw.githubusercontent.com/ehealthplatformstandards/atc-terminology-publisher/main/atc-codesystem.csv"
  "--hl7",
  #"--hl7=6.1.0",
  #"--loinc",
  #"--loinc=2.78",
  #"--snomed=http://snomed.info/sct/11000172109",
  "--snomed=http://snomed.info/sct/11000172109/version/20250315",
  "--extension-country-code=BE"
]
```
This configuration loads:

* The latest **HL7** version
* A specific version of the **SNOMED CT Belgian extension**
* Other terminologies (**ATC**, **BCP13**, ... ) enabled by the `--syndication` flag

LOINC will **not** be imported in this example setup.

📌 **Tips**:
- Adjust ports (80, 8080, 9200) as needed in `docker-compose.yml`.
- Define a volume with bind mounts for externalizing the snomed/loinc and hl7 download location in case the container has a limited storage size.
- Remove `--syndication` to skip startup imports.
- Use `--snomed` with version URI for specific SNOMED versions.

---

## ▶️ Step 4: Start the Server

Launch Snowstorm and its dependencies:

```bash
docker compose up
```

---

## ✅ Step 5: Confirm Import Completion

Check if terminology import has finished:

```bash
curl --location 'localhost:8080/syndication/status?runningOnly=true'
```

An empty list `[]` means the import is complete. Example of an ongoing import response:

```json
[
  {
    "terminology": "hl7",
    "requestedVersion": "latest",
    "status": "RUNNING",
    "timestamp": 1748526433979
  }
]
```

Check if all terminology imports have succeeded:

```bash
curl --location 'localhost:8080/syndication/status?'
```

Example of a response with where Loinc has failed:

```json
[
  {
    "terminology": "hl7",
    "requestedVersion": "latest",
    "actualVersion": "6.3.0",
    "status": "COMPLETED",
    "timestamp": 174852647261
  },
  {
    "terminology": "loinc",
    "requestedVersion": "latest",
    "exception" : "org.springframework.dao.DataAccessResourceFailureException: Connection refused",
    "status": "FAILED",
    "timestamp": 1748526433979
  }
]
```

---

## ✅ Step 6: Validate ATC Terminology

If `--syndication` was used, **ATC** should be imported. Validate with:

<details>
<summary>🧾 cURL Request</summary>

```bash
curl --location 'http://localhost:8080/fhir/CodeSystem/$validate-code?=null' \
--header 'Content-Type: application/json' \
--data '{
  "resourceType": "Parameters",
  "parameter": [
    {
      "name": "coding",
      "valueCoding": {
        "system": "http://whocc.no/atc",
        "code": "J07CA09"
      }
    },
    {
      "name": "displayLanguage",
      "valueString": "en, en-US"
    },
    {
      "name": "default-to-latest-version",
      "valueBoolean": true
    },
    {
      "name": "cache-id",
      "valueId": "b86ac430-2d44-4343-a344-544cab8c36db"
    },
    {
      "name": "includeDesignations",
      "valueBoolean": true
    },
    {
      "name": "diagnostics",
      "valueBoolean": true
    }
  ]
}'
```

</details>

<details>
<summary>📥 Expected Response</summary>

```json
{
  "resourceType": "Parameters",
  "parameter": [
    {
      "name": "result",
      "valueBoolean": true
    },
    {
      "name": "code",
      "valueCode": "J07CA09"
    },
    {
      "name": "system",
      "valueUri": "http://whocc.no/atc"
    },
    {
      "name": "version",
      "valueString": "0"
    },
    {
      "name": "display",
      "valueString": "diphtheria-haemophilus influenzae B-pertussis-poliomyelitis-tetanus-hepatitis B"
    }
  ]
}
```

</details>

---

## 🧪 Postman Collection

You can use this Postman collection to test your Snowstorm instance:

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/2126838-a60204ed-27e8-4c12-8358-ea58be02a431?action=collection%2Ffork&source=rip_markdown&collection-url=entityId%3D2126838-a60204ed-27e8-4c12-8358-ea58be02a431%26entityType%3Dcollection%26workspaceId%3D2e13e762-0976-4818-a6d8-07850f2523ad)

---

## 📌 Legal Notice

By using Snowstorm, you acknowledge and agree to the **terms and conditions** of each terminology source you import or use.

---

## 💡 Tips

* If you modify the `.env` file **after running** `docker compose up`, you must first run `docker compose down` and then `docker compose up` again.
  Simply restarting the affected container will **not** apply the updated environment variables.

* If a **newer version** of a terminology becomes available after you’ve already imported it and you want to upgrade, you have two options:
  – Restart the Snowstorm container
  – Trigger an update using the [**syndication endpoint**](syndication/syndication-on-runtime.md)

* You may also set up a **CRON job** that periodically calls the syndication endpoint using `curl` to ensure you’re always using the most up-to-date terminologies.

---

## 🐛 Debugging

The logs of the Snowstorm and Elasticsearch containers are essential for troubleshooting. Here are some common issues and how to address them:

* **Elasticsearch exited with code 137**
  → The container exceeded the Docker memory limit.
  💡 **Solution**: Increase the memory allocation to **at least 8 GB**.

* **LOINC import times out after 30 seconds**
  → The download from the LOINC website failed.
  This could be due to **invalid credentials** (provided via environment variables), or because the **LOINC site is down or has changed**.
  💡 **Workaround**: If the issue persists using correct credentials, use the **local import** option via the [**syndication endpoint**](syndication/syndication-on-runtime.md).

* **Branch ... is already locked**
  → This typically happens if the import process crashed or was interrupted before it could unlock the CodeSystem branch.
  💡 **Solution**: Prune the Elasticsearch volume and reimport the terminologies.
