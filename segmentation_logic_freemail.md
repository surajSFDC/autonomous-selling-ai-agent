# Salesforce Query and Logic: Filter Out Freemail Domains

This document outlines the SOQL query and associated logic to identify Contacts in Salesforce whose email addresses are from known freemail domains (e.g., Gmail, Yahoo, Outlook.com).

## 1. Target Salesforce Object

*   **Object API Name:** `Contact`

## 2. Relevant Fields

*   **Primary Field for Query:** `Is_Freemail__c`
    *   **Type:** Boolean (This is a custom Formula field on the Contact object, as previously defined in `salesforce_object_outline.md`).
    *   **Purpose:** This field evaluates the standard `Email` field on the Contact. It returns `TRUE` if the email's domain is recognized as a freemail provider and `FALSE` otherwise.

*   **Secondary Field for Context/Output:** `Email`
    *   **Type:** Email (Standard field on the Contact object).
    *   **Purpose:** Stores the actual email address of the contact. This is useful to return alongside the `Id` for verification or display.

## 3. SOQL Query Structure

The query will select Contact records where the `Is_Freemail__c` formula field evaluates to `TRUE`. This directly filters for contacts using freemail addresses.

```soql
SELECT
    Id,         -- Standard Contact ID
    Email,      -- Standard Email field
    Is_Freemail__c, -- Custom formula field, should be TRUE
    AccountId,  -- Optional: To see which Account the Contact belongs to
    FirstName,  -- Optional: For more context
    LastName    -- Optional: For more context
FROM
    Contact
WHERE
    Is_Freemail__c = TRUE
ORDER BY
    AccountId, LastName, FirstName -- Optional: Provides a sorted list of results
```

## 4. Assumption for `Is_Freemail__c` Formula Field

*   **Critical Assumption:** This query's accuracy is entirely dependent on the correct implementation and ongoing maintenance of the `Is_Freemail__c` custom formula field on the `Contact` object.
*   The formula logic within `Is_Freemail__c` is responsible for identifying the domain from the `Email` field and comparing it against a comprehensive list of known freemail provider domains.
*   **Conceptual Example of `Is_Freemail__c` formula logic:**
    ```salesforceformula
    /*
    This formula checks if the domain part of an email address
    belongs to a list of common freemail providers.
    NOTE: Salesforce formula capabilities for robust domain extraction are limited.
    A more robust solution might involve a trigger/flow if complex parsing is needed,
    but for common cases, CONTAINS can work.
    */
    OR(
        CONTAINS(LOWER(Email), "@gmail."),
        CONTAINS(LOWER(Email), "@yahoo."),
        CONTAINS(LOWER(Email), "@outlook."),
        CONTAINS(LOWER(Email), "@hotmail."),
        CONTAINS(LOWER(Email), "@aol."),
        CONTAINS(LOWER(Email), "@icloud."),
        CONTAINS(LOWER(Email), "@zoho."),
        CONTAINS(LOWER(Email), "@gmx."),
        CONTAINS(LOWER(Email), "@protonmail."),
        CONTAINS(LOWER(Email), "@mail.com")
        /* Add other freemail domains as needed, ensuring to check regional variations
           e.g., @yahoo.co.uk, @gmail.co.in etc.
           Consider character limits for formula fields.
        */
    )
    ```
    It's important that this list of freemail domains within the formula is kept reasonably up-to-date.

## 5. Expected Output

The query will return a list of `Contact` records that are identified as having email addresses from freemail providers.

**Example JSON Output (when querying for Id, Email, Is_Freemail__c, AccountId via REST API):**

```json
{
  "totalSize": 2, // Example number of matching records
  "done": true,
  "records": [
    {
      "attributes": {
        "type": "Contact",
        "url": "/services/data/vXX.X/sobjects/Contact/0035j00000ABCDEFGH"
      },
      "Id": "0035j00000ABCDEFGH",
      "Email": "john.doe@gmail.com",
      "Is_Freemail__c": true,
      "AccountId": "0015j00000XYZ123AB",
      "FirstName": "John",
      "LastName": "Doe"
    },
    {
      "attributes": {
        "type": "Contact",
        "url": "/services/data/vXX.X/sobjects/Contact/0035j00000IJKLMNOP"
      },
      "Id": "0035j00000IJKLMNOP",
      "Email": "jane.smith@outlook.com",
      "Is_Freemail__c": true,
      "AccountId": "0015j00000XYZ456CD",
      "FirstName": "Jane",
      "LastName": "Smith"
    }
    // ... other Contact records matching the criteria
  ]
}
```

If the query is simplified to `SELECT Id, Email FROM Contact WHERE Is_Freemail__c = TRUE`, the output for each record in the list would be:
```json
[
  {"Id": "0035j00000ABCDEFGH", "Email": "john.doe@gmail.com"},
  {"Id": "0035j00000IJKLMNOP", "Email": "jane.smith@outlook.com"}
  // ... other Contact records
]
```

This query provides an efficient method to segment contacts, assuming the underlying `Is_Freemail__c` field is well-maintained.
