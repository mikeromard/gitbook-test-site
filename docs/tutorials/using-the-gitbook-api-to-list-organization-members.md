---
description: >-
  This is an example of a python script that lists the display name, email
  address, role, and last seen date for each member in your GitBook
  organization.
tags:
  - tag: python
    primary: true
  - gitbook
---

# Using the GitBook API to list organization members

When I was administering an organization in GitBook, I found it useful to be able to see who my members were, what there roles were, and when they last logged in to GitBook. This helped me to ensure that users had the correct role, and to remove inactive users.

## Prerequisites

* Install [Python](https://www.python.org/) and the [requests module](https://requests.readthedocs.io/en/latest/user/install/#install).
* Create a [personal access token](https://gitbook.com/docs/developers/gitbook-api/quickstart#create-a-personal-access-token) (PAT).
*   Export your PAT and your organization ID as environment variables<br>

    ```shellscript
    export GITBOOK_TOKEN=<your_PAT>
    export ORG_ID=<your_GitBook_organization_ID>
    ```

## The script

In this script I'm using the requests module to make a call to GitBook's [List all organization members](https://gitbook.com/docs/developers/gitbook-api/api-reference/organizations/organization-members#get-orgs-organizationid-members) endpoint, and the [os](https://docs.python.org/3.14/library/os.html#module-os) module to call the environment variables I set in the prerequisites.

{% code title="get_members.py" %}
```python
import requests
import os

gitbook_token = os.environ['GITBOOK_TOKEN']
org_id = os.environ['ORG_ID']
get_members_url = f"https://api.gitbook.com/v1/orgs/{org_id}/members"

response = requests.get(
    get_members_url,
    headers={"Authorization":f"Bearer {gitbook_token}","Accept":"*/*"},
)

data = response.json()

print("name\temail\trole\tlast seen")
for member in data["items"]:
    name = member["user"]["displayName"]
    email = member["user"]["email"]
    role = member["role"]
    last_seen = member["lastSeenAt"]
    print(f'{name}\t{email}\t{role}\t{last_seen}')
```
{% endcode %}

### Running the script

To run this script, I type `python3 get_members.py` in the directory where I've saved it.

### Example output

Here's an example of the output from the script:

{% code title="" %}
```shellscript
name    email   role    last seen
Mike Romard     mromard@gmail.com       admin   2026-07-02T14:53:57.000Z
First Lastname  flastname@example.com   edit    2026-03-03T11:57:24.000Z
Username Smith   usmith@example.com    read    2023-01-01T13:04:56.000Z

```
{% endcode %}

From this, I can see that Username Smith hasn't logged in since January 1, 2023. I would look to see if they still need their account, and remove them if they don't. They may have moved to a different role and no longer need to contribute to our docs, or they may even have left the company.

First Lastname hasn't logged in for almost four months, so I would also check to see if they still need the editor role, or if lower permissions would be more appropriate. It may be that they do still need to be able to edit our docs, but they don't do so frequently.
