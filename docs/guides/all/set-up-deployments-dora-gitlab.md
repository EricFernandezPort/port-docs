---
sidebar_position: 11
displayed_sidebar: null
---

import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"

# Set up deployments using GitLab merge requests or jobs

This guide adapts the [Create & track DORA metrics in your portal](/guides/all/create-and-track-dora-metrics-in-your-portal) guide by demonstrating how to define deployments in Port using GitLab's Merge Requests (MRs) and Jobs.
By the end of this guide, you’ll be able to leverage DORA metrics using GitLab and gain accurate deployment insights in your portal. 

## Prerequisites

To set up GitLab merge requests or jobs for DORA metrics in your portal, you will need:

- Admin permissions (in order to install the experience and execute self-service actions).
- An installed DORA Metrics experience - based on this guide: [Create & track DORA metrics in your portal](/guides/all/create-and-track-dora-metrics-in-your-portal)
- An installed GitLab integration.


## Configure your deployments

1. **Map merge requests and/or jobs as deployments**
    - Go to the **Builder** -> **Data Sources** then click on the GitLab integration
    - Add a new mapping to ingest GitLab merge requests as deployment entities.

:::info Technical note: how deployments are mapped
Keep in mind that the mapping ingests the deployments into the `dora_deployment_event` blueprint, for further information visit [Create & track DORA metrics in your portal](/guides/all/create-and-track-dora-metrics-in-your-portal).
:::

    ```yaml showLineNumbers
    - kind: merge-request
      selector:
        query: 'true'
      port:
        entity:
          mappings:
            identifier: .id | tostring
            title: .title
            blueprint: '"dora_deployment_event"'
            properties:
              deployment_type: '"MR merged"'
              deployment_time: .merged_at
              leadTimeHours: >-
                (.created_at as $createdAt | .merged_at as $mergedAt | ($createdAt
                | sub("\\..*Z$"; "Z") | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime)
                as $createdTimestamp | ($mergedAt | if . == null then null else
                sub("\\..*Z$"; "Z") | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime end)
                as $mergedTimestamp | if $mergedTimestamp == null then null else
                (((($mergedTimestamp - $createdTimestamp) / 3600) * 100 | floor) /
                100) end)
    ```

    - Alternatively, map jobs as deployments:
    ```yaml showLineNumbers
      - kind: job
        selector:
        query: 'true'
        port:
        entity:
            mappings:
            identifier: .id | tostring
            title: .title // (.id | tostring)
            blueprint: '"dora_deployment_event"'
            properties:
                deployment_type: '"GitLab Job"'
                deployment_time: .finished_at
                status: .status
                leadTimeHours: >-
                (.commit.committed_date as $createdAt | .finished_at as $mergedAt |
                ($createdAt | sub("\\.\\d+(Z|\\+00:00)$"; "Z") | 
                strptime("%Y-%m-%dT%H:%M:%SZ") | mktime) as $createdTimestamp | 
                ($mergedAt | if . == null then null else sub("\\.\\d+(Z|\\+00:00)$"; "Z")
                | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime end) as $mergedTimestamp |
                if $mergedTimestamp == null then null else 
                (((($mergedTimestamp - $createdTimestamp) / 3600) * 100 | floor) / 100) end)
    ```

    - Save the mapping. Port will begin ingesting matching events as deployments.

2. **(Optional) Filtering or tagging**
    - You can include/exclude specific branches, labels, or repos in your mapping configuration.
    - Example: Only count deployments that their target branch is `main` and their state is `closed`.
        Using the query filter:
            ```yaml showLineNumbers
        query: ([.state == "closed"] | any) and ([.target_branch == "main"] | any)
        ```

3. **Test the configuration**
    - Trigger a new merge request and complete it, or run a deployment job.
    - In Port, navigate to the relevant deployment blueprint and verify the new deployment appears.


## Next steps

- View your DORA metrics in the dashboard for deployment insights once integration data is ingested.
- Set up incident tracking to enable full DORA coverage.
- Link deployments to teams or services for fine-grained metrics.

<img src='/img/guides/doraDashboardExample.png' width='80%' border='1px' />