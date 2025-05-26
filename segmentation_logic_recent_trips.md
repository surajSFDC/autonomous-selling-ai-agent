# Salesforce Query and Logic: Organizations Without Recent Trips

This document outlines the SOQL query and associated logic to identify organizations (Accounts) in Salesforce that have not had a "trip" (a significant activity tracked by `Last_Activity_Date__c`) within the last 2 days.

## 1. Target Salesforce Object

*   **Object API Name:** `Account`

## 2. Relevant Field

*   **Field API Name:** `Last_Activity_Date__c`
*   **Type:** DateTime (Custom field on the Account object)
*   **Purpose:** Stores the timestamp of the organization's last recorded "trip" or significant engagement.

## 3. SOQL Query Structure

The query selects Account records where the `Last_Activity_Date__c` is either:
    a) `NULL` (indicating no trip has ever been recorded).
    b) Earlier than the dynamically calculated `cutoffDateTime` (more than 2 days ago).

```soql
SELECT
    Id,
    Name,
    Last_Activity_Date__c
FROM
    Account
WHERE
    Last_Activity_Date__c = NULL OR Last_Activity_Date__c < :cutoffDateTime
ORDER BY
    Last_Activity_Date__c DESC NULLS FIRST, Name
```

*   `:cutoffDateTime`: This is a bind variable representing the calculated date and time threshold (see section 4).
*   `ORDER BY`: Optional, but useful for reviewing results. `NULLS FIRST` ensures accounts with no activity appear at the top.

## 4. Dynamic Calculation of "2 days ago" Cutoff (`cutoffDateTime`)

The `cutoffDateTime` variable must be calculated by the system executing the SOQL query *before* the query is run.

**Key Principle:** The calculation should result in a DateTime value. For example, if today is `2024-03-10T15:00:00Z`, then "2 days ago" would target any `Last_Activity_Date__c` before `2024-03-08T15:00:00Z`. If the intention is to include any activity *before* the start of the day 2 days ago, the time component should be set to `00:00:00Z`.

**Example Calculation Logic (Pseudocode):**

```
current_datetime = getCurrentUTCDateTime() // It's crucial to use UTC for consistency with Salesforce
cutoff_datetime = current_datetime.subtract(Duration.days(2))
// To set to the beginning of that day (if needed):
// cutoff_datetime = date_from_datetime(current_datetime.subtract(Duration.days(2))) + Time(0,0,0)
```

**Implementation Examples:**

*   **Apex (Salesforce):**
    ```apex
    // To get a DateTime 2 days ago from now:
    DateTime cutoffDateTime = System.now().addDays(-2);

    // To get a DateTime representing the START of the day 2 days ago:
    // Date twoDaysAgoDate = System.today().addDays(-2);
    // DateTime cutoffDateTime = DateTime.newInstance(twoDaysAgoDate, Time.newInstance(0, 0, 0, 0));

    List<Account> inactiveAccounts = [
        SELECT Id, Name, Last_Activity_Date__c
        FROM Account
        WHERE Last_Activity_Date__c = NULL OR Last_Activity_Date__c < :cutoffDateTime
        ORDER BY Last_Activity_Date__c DESC NULLS FIRST, Name
    ];
    ```

*   **Python (External System using Salesforce API):**
    ```python
    from datetime import datetime, timedelta, timezone

    # Current UTC datetime
    now_utc = datetime.now(timezone.utc)
    # Cutoff datetime (2 days ago from now)
    cutoff_datetime_val = now_utc - timedelta(days=2)

    # To use the start of the day 2 days ago (UTC):
    # two_days_ago_date = (now_utc - timedelta(days=2)).date()
    # cutoff_datetime_val = datetime(two_days_ago_date.year, two_days_ago_date.month, two_days_ago_date.day, 0, 0, 0, tzinfo=timezone.utc)

    # Format for SOQL (ISO 8601)
    cutoff_datetime_soql = cutoff_datetime_val.strftime('%Y-%m-%dT%H:%M:%SZ')

    soql_query = f"SELECT Id, Name, Last_Activity_Date__c FROM Account WHERE Last_Activity_Date__c = NULL OR Last_Activity_Date__c < {cutoff_datetime_soql} ORDER BY Last_Activity_Date__c DESC NULLS FIRST, Name"
    # ... then execute query via API
    ```

## 5. Expected Output

The query will return a list of `Account` records that meet the criteria. Each record will, at a minimum, contain the `Id`. Including `Name` and `Last_Activity_Date__c` is recommended for context.

**Example JSON Output (if queried via REST API):**

```json
{
  "totalSize": 2,
  "done": true,
  "records": [
    {
      "attributes": {
        "type": "Account",
        "url": "/services/data/vXX.X/sobjects/Account/001R000000abcdefgh"
      },
      "Id": "001R000000abcdefgh",
      "Name": "Never Active Corp",
      "Last_Activity_Date__c": null
    },
    {
      "attributes": {
        "type": "Account",
        "url": "/services/data/vXX.X/sobjects/Account/001R000000ijklmnop"
      },
      "Id": "001R000000ijklmnop",
      "Name": "Long Ago Systems",
      "Last_Activity_Date__c": "2024-01-15T11:00:00.000+0000"
    }
  ]
}
```

If only Account IDs are needed, the query can be `SELECT Id FROM ...`, and the output would be simpler, e.g., a list of strings:
`["001R000000abcdefgh", "001R000000ijklmnop"]`
