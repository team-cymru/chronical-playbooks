# Google SecOps Playbooks

This repository contains the sample playbooks that uses the **TeamCymruScout** integration in **Google SecOps SOAR** platform to gather intel and perform steps based on this intel.

These automated playbooks can be used as a reference to generate a custom workflow to fulfill your needs.

## Import Playbooks

Importing a playbook into any SecOps instance is a very straight-forward task:

- Download the `.zip` file of the playbook that needs to be imported. This file would also contain all the blocks that are used in the playbook.
  ![Download ZIP](<./screenshots/Download ZIP.png>)

- Open the SecOps instance. From the sidebar, navigate to **Response > Playbooks** section.
  ![Sidebar - Playbooks](<./screenshots/Playbook Section.png>)

- Click on the **three dots** icon at the top of the Playbooks, and then click on **Import**.
  ![Import Playbook](<./screenshots/Import Playbook.png>)

- Select the `.zip` file downloaded earlier.

The playbooks stored in the file would be imported. We can access these under the **Imported Playbooks** folder. These playbooks and blocks can be moved to any other folder of our choice.

## Documentation

- [Playbooks](#playbooks)
  - [Block Suspicious IP](#block-suspicious-ip)
  - [Cache IP Details](#cache-ip-details)
  - [Get Domain Certs](#get-domain-certs)
  - [Suspicious Tags](#suspicious-tags)
  - [Playbook with Views](#playbook-with-views)
  - [Enrich Peers Info](#enrich-peers-info)
- [Blocks](#blocks)
  - [Is IP Suspicious](#is-ip-suspicious)
  - [Block IP](#block-ip)
  - [Time difference](#time-difference)
- [Known Issues](#known-issues)

---

### Playbooks

Playbooks are a feature in Google Security Operations (SecOps) that can be used to automatically perform actions on alerts/cases when triggered. Playbooks can be used to: Retrieve information about alerts, Remediate threats, Create tickets, and Manage toxic combinations and IAM recommendations.

**Points to Note:**

- `.zip` file for every playbook contains the playbook along with all the blocks used in that playbook.
- Each playbook requires a trigger, which decides when the playbook would be run. For all the playbooks created, the Trigger is set as **All**. Users can update the Trigger conditions based on the requirements.
- As the trigger of every playbook is set to run on all ingested alerts into SecOps instance, the imported playbooks are disabled by default. User can easily enable any playbook by simply cliking on the toggle button beside the playbook name. ([Reference](https://cloud.google.com/chronicle/docs/soar/respond/working-with-playbooks/whats-on-the-playbooks-screen#:~:text=At%20the%20top%20segment%20of%20the%20playbook%20designer%20pane%2C%20you%20can%20use%20the%20horizontal%20toggling%20button%20to%20enable%20or%20disable%20the%20playbook.))

#### Block Suspicious IP

This playbook uses TeamCymruScout's intelligence to determine the maliciousness of an input IP address, and based on this information, the case is closed, or raised to an incident.

![Playbook - Block Suspicious IP](<./screenshots/playbooks/Block Suspicious IP.png>)

**Flow**:

- Is IP Suspicious (block)

  - This block checks whether the given IP address is malicious or not. More details can be found [here](#is-ip-suspicious).

- Is IP Suspicious ? (condition)

  - This is a condition that determines if the previous block (Is IP Suspicious) returned output as `true` or not.
  - If the output is `true`, the branch with Block IP block would be executed. Otherwise, the branch to close the case would be executed.

- Block IP (block)

  - This block would be executed in case the `Is IP Suspicious ?` condition is `true`. More details can be found [here](#block-ip).

- Priority - Informative (action)

  - This action (Change Priority) is provided by SecOps with the Siemplify integration. It updates the priority of the current case to Informative, meaning that the case can be taken up with the least priority.
  - This action would be executed in case the `Is IP Suspicious ?` condition is `false`.

- Close Case (action)
  - This action is provided by SecOps with the Siemplify integration. It simply closes the case on which this playbook is run, with an appropriate comment.
  - This action would be executed after the `Priority - Informative` action is executed.

---

#### Cache IP Details

This playbook uses **custom lists** as a cache storage for storing the details of IP addresses fetched from Team Cymru Scout platform. In case the data is already cached, no API calls would be made. If the data is not cached, or if the cache is expired (24 hours), an API call will be made to Team Cymru Scout and the data will be cached.

![Playbook - Cache IP Details](<./screenshots/playbooks/Cache IP Details.png>)

**Flow**:

- Retrieve IP details from cache (action)

  - This action (Search Custom Lists) is a part of [Lists power-up](https://cloud.google.com/chronicle/docs/soar/marketplace/power-ups/lists#search-custom-list) provided by SecOps.
  - It searches whether an input text is present in any record of a custom list. Currently, we use the custom list named **TCS_ips** to store the IP details.
  - In case the text is present in any record of the custom list, the output of the action would be `true`. Otherwise, the output would be `false`. Also, a JSON output containing the matched record will be return as output.

- Are there matches ? (condition)

  - This is a condition that determines if the previous action (Retrieve IP details from cache) returned output as `true` or not.
  - If the output of this condition is `true`, it means that the data for the IP has already been cached. However, it is not confirmed if this data is stale or not. In case the condition is not true, we must fetch the data from Team Cymru Scout.

- Time difference (block)

  - This block computes the time difference between the current time and the last time the IP details were cached. More details can be found [here](#time-difference).
  - This block will be executed only if the previous condition `Are there matches ?` is `true`.

- Less than 24H (condition)

  - This condition uses the output from the previous block (Time difference) and checks whether the time difference is less than 24 hours (24 * 60 * 60 seconds).
  - If the condition is `true`, we can simply output the stored JSON data. Otherwise, we must delete the existing data, as it is stale, and fetch the data from Team Cymru Scout. This latest data will be cached again.

- Display JSON Result (action)

  - This action (Buffer) is a part of [Tools power-up](https://cloud.google.com/chronicle/docs/soar/marketplace/power-ups/tools#buffer) provided by SecOps. It can display output in a JSON format.
  - This action would be run only if the cached data is not stale (`Less than 24H` condition is `true`). It simply displays the input (cached data) as a JSON output.

- Remove Old Record (action)

  - This action (Remove String from Custom List) is a part of [Lists power-up](https://cloud.google.com/chronicle/docs/soar/marketplace/power-ups/lists#remove-string-from-custom-list) provided by SecOps. It removes a string from a custom list.
  - This action would be run only if the cached data is stale (`Less than 24H` condition is `false`). In this case, we would remove the cached data from the custom list.

- Get Fresh Record from Team Cymru Scout (action)

  - This action (Get IP Details) is a part of TeamCymruScout integration. It expects a comma-separated string of IP addresses as input, and returns details information about these IPs in JSON format.
  - In this playbook, we would run this and the subsequent actions only if any one of these conditions:
    - The condition `Are there matches ?` is `false` (Cache miss).
    - The condition `Less than 24H` is `false` (Cache is stale).

- Run JSONPath Query (action)

  - This action is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#run-jsonpath-query) provided by SecOps. It can extract information out of a JSON input using JSONPath expressions.
  - From the API response received by the previous action (Get Fresh Record from Team Cymru Scout), this action extracts the `ip` value of the input IP.

- Cache new Record (action)
  - This action (Add String to Custom List) is a part of [Lists power-up](https://cloud.google.com/chronicle/docs/soar/marketplace/power-ups/lists#add-string-to-custom-list) provided by SecOps. It adds a string entry into a custom list.
  - Once the IP address and its details are fetched from Team Cymru Scout, we would cache the data in the custom list, for further use.

---

#### Get Domain Certs

This playbook allows users to get information about the certificates assigned to the IP addresses that are associated with a given domain.

![Playbook - Get Domain Certs](<./screenshots/playbooks/Get Domain Certs.png>)

**Flow**:

- PDNS Domain Query (action)

  - This action (Advanced Query) is a part of TeamCymruScout integration.
  - It expects a domain name that falls within the convention of [Scout Query Language](https://scout.cymru.com/docs/scout/ultimate#help.scout.index-panel).
  - This action returns the details of the IPs that match the query passed as input.

- Display Certificates (action)
  - This action (Buffer) is a part of [Tools power-up](https://cloud.google.com/chronicle/docs/soar/marketplace/power-ups/tools#buffer) provided by SecOps. It can display output in a JSON format.
  - This action uses the response generated by the previous action (PDNS Domain Query) and extracts the certificate information for the IPs. This certificate information will be displayed as JSON output.

---

#### Suspicious Tags

This playbook allows users to enter the tags they consider suspicious, and if the given IP address has any of the associated tags that match these suspicious tags, then they can block this IP.

**NOTE:** The action Extract Tags used in this playbook strictly supports only a single IP address.

![Playbook - Suspicious Tags](<./screenshots/playbooks/Suspicious Tags.png>)

**Blocks Used:** [Block IP](#block-ip)

**Flow**:

- Extract Tags for IP (action)

  - This action (Extract Tags) is a part of TeamCymruScout integration.
  - It expects a single IP address and Tags as inputs. From Team Cymru Scout platform, the tags associated with the given IP, and its peers would be fetched and compared with the provided tags. In case any match is found, the action would output `true`. If no match is found, the output would be `false`.

- Matching Tags Found ? (condition)

  - This is a condition that checks if the previous action (Extract Tags for IP) is `true` or `false`.

- Block IP (block)

  - This block would be executed in case the `Matching Tags Found ?` condition is `true`. More details can be found [here](#block-ip).

- Close Case (action)
  - This action is provided by SecOps with the Siemplify integration. It simply closes the case on which this playbook is run, with an appropriate comment.
  - This action would be executed in case the `Matching Tags Found ?` condition is `false`.

#### Playbook with Views

This playbook enriches all the IP entities present in the alert and generates a set of widgets (view) that can be viewed on the Alert Overview page.

![Playbook - Playbook with Views](<./screenshots/playbooks/Playbook with Views.png>)

**Flow**:

- Enrich IPs (action)
  - This action (Enrich IPs) is a part of TeamCymruScout integration.
  - This action collects all the IP entities and enriches them with their details from Team Cymru Scout. The enriched data include the rating and tags for each IP entity.

**View**

![Views - Playbook with Views](<./screenshots/playbooks/Views - Playbook with Views.png>)

- Team Cymru Playbook
  - **Enriched IPs - Team Cymru (JSON)**: Displays the JSON response received from Team Cymru Scout platform for the IP entities.
  - **Entities Highlights**: Highlights all the entities present in the alert. For each entity, user can views the details and the fields enriched by TeamCymruScout integration (fields start with TCS_).
  - **Enriched IPs (HTML - Table)**: This widget displays the IP entities along with their ratings and last enriched time, in a tabular format.

#### Enrich Peers Info

This playbook enriches all the IP entities present in the alert and generates an HTML table widget (view) that can be viewed on the Alert Overview page.

![Playbook - Enrich Peers Info](<./screenshots/playbooks/Enrich Peers Info.png>)

**Flow**:

- Get Peers Info (action)
  - This action (Get Peers Info) is a part of TeamCymruScout integration.
  - This action collects all the IP entities and fetches the detailed peers information for each IP entity from Team Cymru Scout.

**View**

![View - Enrich Peers Info](<./screenshots/playbooks/Views - Enrich Peers Info.png>)

- Peers Info
  - **Peers Info - Team Cymru Scout (HTML - Table)**: This widget displays the IP entities along with their peer information, in a tabular format. The details include:
    - Source IP address
    - Peer IP address
    - Event count
    - Protcol used for connection with each peer IP
    - Tags associated with peer IP address
    - Server/dest ports used for connection with peer IP

---

### Blocks

A block is a re-usable set of actions and conditions that can be used in multiple playbooks. This acts as a wrapper for performing some set of actions, that are often performed in multiple playbooks.

#### Is IP Suspicious

This block takes multiple IPs as input, and based on the overall rating received from Team Cymru Scout, of the first IP, it outputs `true` or `false`. In case the overall rating is `malicious` or `suspicious`, the output would be `true`. In other cases, output would be `false`.

![Block - Is IP Suspicious](<./screenshots/blocks/Is IP Suspicious.png>)

**Input:**

IP Addresses - Comma-separated string of IP Addresses.

**Flow:**

- List IP Summary (action)

  - This action is a part of TeamCymruScout integration. It accepts a comma-separated string of IPs and a Limit parameter.
  - The summary information for each IP would be fetched and the cumulative response will be returned as JSON output.

- Extract Overall Ratings (action)

  - This action is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#run-jsonpath-query) provided by SecOps. It can extract information out of a JSON input using JSONPath expressions.
  - From the summary information received by the previous action, this action extracts the `overall_rating` value of the IPs.

- Is IP Suspicious ? (condition)
  - This is a condition that determines if the overall rating of the **first** input IP is `malicious` or `suspicious`.
  - If the rating is malicious or suspicious, the output of the block will be `true`. In other cases, the output would be `false`.

**Output:**

`True`, or `False`, depending on the `Is IP Suspicious ?` condition.

---

#### Block IP

This block can be used once an IP is found to be malicious, or suspicious. It raises the case for which the playbook runs, to an incident, notifying the SOC Managers about this. It also creates an incident(ticket) in ServiceNow platform to block and isolate the IP.

**NOTE:** ServiceNow integration is used here as a ticketing platform. Users can use any integration of their choice. Users must install the ServiceNow integration and configure it in order to use this block.

![Block - Block IP](<./screenshots/blocks/Block IP.png>)

**Input:**

IP Address - Suspicious/malicious IP that is to be blocked

**Flow:**

- Change Priority (action)

  - This action is provided by SecOps with the Siemplify integration. It updates the priority of the current case to Critical.

- Raise Incident (action)

  - This action is provided by SecOps with the Siemplify integration. It raises the current case to an Incident. Incidents in SecOps are the positive true cases that need to be looked upon at priority.

- Create ServiceNow Incident (action)
  - This action (Create Incident) is a part of ServiceNow integration. It simply creates an Incident (ticket) in ServiceNow platform, with a description to isolate and block the suspicious IP.

**Output:**

None

---

#### Time difference

This block compares the input timestamp (in milliseconds) with the current time, and returns the time difference in seconds.

![Block - Time difference](<./screenshots/blocks/Time difference.png>)

**Input:**

input_ms_time - Time in milliseconds which is to be compared with current time.

**Flow:**

- Convert ms to seconds (action)

  - This action (Math Arithmetic) is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#math-arithmetic) provided by SecOps. It can perform arithmetic operations on input values.
  - In this block, this action is used to simply convert the input time in milliseconds to seconds.

- Convert seconds to integer (action)

  - This action (Math Functions) is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#math-functions) provided by SecOps. It can perform basic math operations like type conversions, finding absolute value, etc.
  - In this block, this action is used to round the seconds to integer returned in the previous action.

- Convert Time Format (action)

  - This action (Convert Time Format) is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#math-convert-time-format) provided by SecOps. It can convert time from one format to another.
  - In this block, this action is used to convert the seconds to the following datetime format: `YYYY-MM-DD HH:mm:ss`.

- Calculate time difference (action)
  - This action (Time Duration Calculator) is a part of [Functions power-up](https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/power-ups/functions#time-duration-calculator) provided by SecOps. It compares two datetime values and returns the time difference.
  - In this block, this action is used to compare the input time with the current time, and return the results. As a return value, we output the time difference in only **seconds** format, as it provides more flexibility.

**Output:**

Time difference between the input timestamp and current time, in seconds.

### Known Issues

1. Currently, playbooks in Google SecOps do not support for-loop mechanism. Hence, in blocks like [Is IP Suspicious](#is-ip-suspicious), we do get information about multiple IP addresses, but we consider the maliciousness of only the first IP address as the output. The reason for this can be considered using the following example:

    Suppose, for the playbook [Block Suspicious IP](#block-suspicious-ip), ideally, we would want to pass multiple IP addresses in the input. For each IP, based on its maliciousness received from Team Cymru Scout, we should be able to create a ticket in the ServiceNow platform (using the `Block IP` block). But due to the limitation of SecOps, we cannot iterate through the maliciousness of IPs and check the condition `Is IP Suspicious ?`. Hence, we rely on only the first IP address from the list.

2. For actions like `Extract Tags`, we expect only a single IP address. Hence, in case when multiple IPs are passed as input, the action would fail and throw error, indicating that the input is an invalid IP address. This error can be avoided by manipulating the playbook to only pass a single IP address in the input to such actions. The possible inputs for each action of TeamCymruScout integration can be found from the User Guide.

## Troubleshooting

In case of any failures when running any playbook, we can debug the same by running the playbook in the Simulator Mode. For more details, please refer to this guide: [Google SecOps playbooks - Simulator](https://cloud.google.com/chronicle/docs/soar/respond/working-with-playbooks/working-with-playbook-simulator).

## References

- Google SecOps playbooks - [Documentation](https://cloud.google.com/chronicle/docs/secops/google-secops-soar-toc#work-with-playbooks)
- Create views with playbooks - [Documentation](https://cloud.google.com/chronicle/docs/soar/respond/working-with-playbooks/define-customized-alert-views-from-playbook-designer)
- Limitations around for-loop usage in playbooks:
  - [Cycle over a list](https://www.googlecloudcommunity.com/gc/SOAR-Forum/Cycling-over-a-list-in-a-playbook/m-p/642352)
  - [Iterate through a JSON list](https://www.googlecloudcommunity.com/gc/SOAR-Forum/Iterate-through-a-json-list/m-p/821359#M2891)
  - [Looping in Playbooks](https://www.googlecloudcommunity.com/gc/SOAR-Forum/Loop-in-a-PB-through-a-list-fetched-using-http-get-connector/m-p/639073)
  - [Functions that require single input](https://www.googlecloudcommunity.com/gc/SOAR-Forum/Functions-that-require-single-input/m-p/639027)
- TODO: Add link to TeamCymruScout user guide
