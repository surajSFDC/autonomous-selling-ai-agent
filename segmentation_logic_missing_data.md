# Salesforce Query and Logic: Identify and Flag Restaurants Missing Key Data

This document details the SOQL queries and update logic to identify and flag `Account` records (restaurants) that are missing critical data points, such as `Account.Cuisine_Type__c` or `Contact.Phone` for key contacts. This process focuses on Accounts in an active onboarding process.

## 1. Target Record Identification

*   **Primary Target:** `Account` records linked to an "active" `Restaurant_Onboarding__c` record.
    *   An "active" `Restaurant_Onboarding__c` record is defined as one where its `Onboarding_Stage__c` is NOT IN ('Onboarding Complete', 'Onboarding Halted / Cancelled').
*   **Data Scope:** The goal is to identify missing data that is crucial for the progression of the onboarding process.

## 2. Querying for Accounts with Missing Data

We will primarily query the `Account` object and use subqueries or relationship queries to access related `Restaurant_Onboarding__c` and `Contact` information.

*   **Query 1: Accounts Missing `Account.Cuisine_Type__c` OR Phone for `Account.Primary_Contact__c`**
    *   This query identifies accounts that are missing their cuisine type or where their designated primary contact is missing a phone number.
    *   **SOQL:**
        ```soql
        SELECT
            Id,
            Name,
            Cuisine_Type__c,
            Primary_Contact__c,         // ID of the Primary Contact
            Primary_Contact__r.Phone,   // Phone of the Primary Contact (if Primary_Contact__c is populated)
            Primary_Contact__r.Email,   // Email of the Primary Contact for identification
            Missing_Onboarding_Data_Flags__c, // Existing flags on Account
            (SELECT Id, Onboarding_Stage__c, Missing_Information_Detail__c FROM Restaurant_Onboardings__r WHERE Onboarding_Stage__c NOT IN ('Onboarding Complete', 'Onboarding Halted / Cancelled') LIMIT 1) // Subquery for active onboarding record and its details
        FROM
            Account
        WHERE
            (Cuisine_Type__c = NULL OR (Primary_Contact__c != NULL AND Primary_Contact__r.Phone = NULL)) // Condition for missing data
            AND Id IN (
                SELECT Account__c
                FROM Restaurant_Onboarding__c
                WHERE Onboarding_Stage__c NOT IN ('Onboarding Complete', 'Onboarding Halted / Cancelled')
            ) // Ensures Account is linked to an active onboarding process
        ```
        *   `Restaurant_Onboardings__r` is the default child relationship name from `Account` to `Restaurant_Onboarding__c`.
        *   The condition `(Primary_Contact__c != NULL AND Primary_Contact__r.Phone = NULL)` specifically checks for a missing phone only if a primary contact is actually assigned.

*   **Query 2: Identifying Missing Phone for Other Key Contacts (e.g., 'Manager' or 'Owner' Roles)**
    *   This often requires processing after an initial fetch, as direct SOQL for "any contact with role X is missing phone" across many accounts can be complex or hit limits.
    *   **Step 2a: Fetch Accounts with Active Onboarding and Their Key Contacts:**
        ```soql
        SELECT
            Id,
            Name,
            Cuisine_Type__c,
            Missing_Onboarding_Data_Flags__c, // Existing flags on Account
            (SELECT Id, Onboarding_Stage__c, Missing_Information_Detail__c FROM Restaurant_Onboardings__r WHERE Onboarding_Stage__c NOT IN ('Onboarding Complete', 'Onboarding Halted / Cancelled') LIMIT 1), // Active onboarding record
            (SELECT Id, FirstName, LastName, Email, Phone, Role_At_Restaurant__c FROM Contacts WHERE Role_At_Restaurant__c IN ('Owner', 'Manager', 'Primary Decision Maker')) // Subquery for relevant contacts
        FROM
            Account
        WHERE
            Id IN (
                SELECT Account__c
                FROM Restaurant_Onboarding__c
                WHERE Onboarding_Stage__c NOT IN ('Onboarding Complete', 'Onboarding Halted / Cancelled')
            )
        ```
    *   **Step 2b: Process in Application Layer (e.g., Apex, Python AI Backend):**
        Iterate through each `Account` from the results. For each `Account`:
        1.  Check if `Account.Cuisine_Type__c` is NULL.
        2.  Iterate through its related `Contacts` list (from the subquery).
        3.  If any `Contact` with a specified key role (e.g., 'Owner', 'Manager') has `Phone = NULL`, then this `Account` needs flagging for that missing contact phone.

## 3. Flagging Action (Logic for Update)

The AI or an automated process will update a designated field to reflect the missing data.

*   **Target Field for Flags:**
    *   **Primary:** `Account.Missing_Onboarding_Data_Flags__c` (Recommended: Text Area for flexibility, or Multi-select Picklist if flags are predefined and limited).
    *   **Secondary (or in addition):** `Restaurant_Onboarding__c.Missing_Information_Detail__c` (Long Text Area on the onboarding record itself, useful for details specific to that onboarding instance).

*   **Update Logic (Implemented in Apex, Flow, or by the AI's interaction with the Salesforce API):**

    1.  **Retrieve Account(s):** Fetch the Account(s) identified by the queries in Step 2.
    2.  **Determine Missing Data:** For each Account, identify which specific pieces of data are missing (Cuisine Type, Primary Contact Phone, Other Key Contact Phone).
    3.  **Construct Flag Message(s):** Create short, descriptive messages for each missing item.
        *   Example: "Missing Cuisine Type"
        *   Example: "Missing Phone for Primary Contact"
        *   Example: "Missing Phone for Manager: [Contact Name]"
    4.  **Update Target Field:**
        *   Get the current value of `Account.Missing_Onboarding_Data_Flags__c` (or `Restaurant_Onboarding__c.Missing_Information_Detail__c`).
        *   **Append New Flags:** Add the new flag messages to the existing content. Use a consistent delimiter (e.g., semicolon followed by a space "; ") if it's a Text Area.
        *   **Avoid Duplicates:** Before appending, check if the exact flag message already exists in the field to prevent redundancy (e.g., "Missing Cuisine Type; Missing Cuisine Type").
        *   If the field was empty, set it with the new flag messages.
    5.  **Perform DML:** Update the `Account` or `Restaurant_Onboarding__c` record(s).

    **Conceptual Apex Snippet for Appending to a Text Area Field:**
    ```apex
    // Account acc = fetched Account record;
    // List<String> newFlags = list of new flag strings like "Missing Cuisine Type";

    String currentFlags = acc.Missing_Onboarding_Data_Flags__c;
    List<String> flagsToAdd = new List<String>();

    for (String newFlag : newFlags) {
        if (currentFlags == null || !currentFlags.contains(newFlag)) {
            flagsToAdd.add(newFlag);
        }
    }

    if (!flagsToAdd.isEmpty()) {
        String combinedNewFlags = String.join(flagsToAdd, '; ');
        if (String.isBlank(currentFlags)) {
            acc.Missing_Onboarding_Data_Flags__c = combinedNewFlags;
        } else {
            acc.Missing_Onboarding_Data_Flags__c = currentFlags + '; ' + combinedNewFlags;
        }
        // Add acc to a list for DML update
    }
    ```

## 4. Example Output (Updated Flag Field)

Assuming `Account.Missing_Onboarding_Data_Flags__c` is a Text Area field:

*   **Initial State (empty):** `null`
*   **After finding Cuisine Type is missing:** `"Missing Cuisine Type"`
*   **Later, finding Primary Contact Phone also missing (and Cuisine Type still missing):** `"Missing Cuisine Type; Missing Phone for Primary Contact"`
*   **If it previously had "Banking Details Pending" and now Cuisine Type and a Manager's phone are found missing:** `"Banking Details Pending; Missing Cuisine Type; Missing Phone for Manager: Jane Doe"`

This structured flagging helps create a clear action item for data enrichment, either by an internal team member or by the AI assistant initiating a request for information from the restaurant.Okay, I have created the `segmentation_logic_missing_data.md` file.

It includes:
1.  **Target Record Identification:** Focuses on `Account` records linked to an active `Restaurant_Onboarding__c` record (where `Onboarding_Stage__c` is not 'Onboarding Complete' or 'Onboarding Halted / Cancelled').
2.  **Querying for Accounts with Missing Data:**
    *   Provided a SOQL query for `Account` to find instances where `Account.Cuisine_Type__c` is NULL OR `Account.Primary_Contact__r.Phone` is NULL, while ensuring the Account is linked to an active onboarding process using a subquery on `Restaurant_Onboarding__c`.
    *   Outlined a two-step approach for identifying missing phone numbers for other key contacts (e.g., 'Manager', 'Owner' roles): first, query Accounts with their related key contacts, then process these results in an application layer (like Apex or the AI backend) to check for missing phone numbers among those contacts.
3.  **Flagging Action (Logic for Update):**
    *   Specified `Account.Missing_Onboarding_Data_Flags__c` (Text Area recommended for flexibility) as the primary target field for flags, with `Restaurant_Onboarding__c.Missing_Information_Detail__c` as an alternative or additional field.
    *   Described the update logic: retrieve the Account, determine missing data, construct flag messages (e.g., "Missing Cuisine Type", "Missing Phone for Primary Contact"), and append these new, unique flags to the existing content of the target field using a delimiter (e.g., semicolon). Provided conceptual Apex code for appending flags.
4.  **Example Output:** Showed how the `Account.Missing_Onboarding_Data_Flags__c` field might look with one or more appended flags, including how it combines with pre-existing flags.

This document should provide a clear guide for implementing the required segmentation and flagging logic.
