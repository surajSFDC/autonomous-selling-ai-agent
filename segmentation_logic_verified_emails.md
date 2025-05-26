# Salesforce Query and Logic: Identify Accounts with Verified Emails

This document outlines the SOQL query and associated logic to identify Accounts in Salesforce that are marked as having verified email addresses at an organizational level. This is based on the custom field `Verified_Email_Organization__c`.

## 1. Target Salesforce Object

*   **Object API Name:** `Account`

## 2. Relevant Field

*   **Field API Name:** `Verified_Email_Organization__c`
    *   **Type:** Boolean (Custom field on the Account object, as defined in `salesforce_object_outline.md`).
    *   **Purpose:** When `TRUE`, this field indicates that the organization (Account) has been confirmed to have its email information verified. This typically means at least one key contact associated with the Account has a validated, non-freemail email address. A value of `FALSE` or `NULL` would indicate the organization's email status is not verified or is unknown.

## 3. SOQL Query Structure

The query will select Account records where the `Verified_Email_Organization__c` field is explicitly set to `TRUE`.

```soql
SELECT
    Id,    -- Standard Account ID
    Name,  -- Optional: Standard Account Name, for context
    Verified_Email_Organization__c -- Optional: To confirm the field's value in results
FROM
    Account
WHERE
    Verified_Email_Organization__c = TRUE
ORDER BY
    Name -- Optional: Provides a sorted list of results by Account Name
```

## 4. Assumption for `Verified_Email_Organization__c` Population

*   **Critical Assumption:** The utility of this query is entirely dependent on the `Verified_Email_Organization__c` field being accurately populated and maintained by a separate, reliable process.
*   The AI assistant, for this specific segmentation, consumes the value of this field rather than performing the verification itself.
*   This external process might involve:
    *   **Automated Logic:** A batch Apex job, a Salesforce Flow, or an integration with an external email validation service. This automation would typically check the status of primary Contact(s) linked to the Account. For instance, if a primary Contact has their email address verified (and it's not a freemail address, i.e., `Contact.Is_Freemail__c = FALSE`), the `Account.Verified_Email_Organization__c` field could be set to `TRUE`.
    *   **Manual Updates:** Sales, support, or data stewardship teams might manually update this field based on their direct interactions, successful email communications, or other verification methods.
*   The definition of "verified" (e.g., email bounced check, link clicked in an email, successful reply) should be consistently applied by the process that populates this field.

## 5. Expected Output

The query will return a list of `Account` records that are marked as having verified email addresses at the organizational level.

**Example JSON Output (when querying for Id, Name, and Verified_Email_Organization__c via REST API):**

```json
{
  "totalSize": 2, // Example number of matching records
  "done": true,
  "records": [
    {
      "attributes": {
        "type": "Account",
        "url": "/services/data/vXX.X/sobjects/Account/0015j00000VALIDATE1"
      },
      "Id": "0015j00000VALIDATE1",
      "Name": "Trusted Partners Inc.",
      "Verified_Email_Organization__c": true
    },
    {
      "attributes": {
        "type": "Account",
        "url": "/services/data/vXX.X/sobjects/Account/0015j00000VERIFYNOW2"
      },
      "Id": "0015j00000VERIFYNOW2",
      "Name": "SecureComms Ltd.",
      "Verified_Email_Organization__c": true
    }
    // ... other Account records matching the criteria
  ]
}
```

If the query is simplified to `SELECT Id FROM Account WHERE Verified_Email_Organization__c = TRUE`, the output will be a list of Account IDs:
```json
[
  "0015j00000VALIDATE1",
  "0015j00000VERIFYNOW2"
  // ... additional Account IDs
]
```

This query provides a direct method to segment Accounts based on their email verification status, relying on the pre-established `Verified_Email_Organization__c` flag.
