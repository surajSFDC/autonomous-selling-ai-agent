# Salesforce Query and Logic: New Restaurant Sign-ups in Last 24 Hours

This document details the SOQL queries and associated logic to identify new restaurant sign-ups within Salesforce from the last 24 hours. This includes newly created `Lead` records and newly created `Account` records that are in a "Prospecting" or initial sign-up stage.

## 1. Query for New Leads

*   **Object:** `Lead`
*   **Purpose:** To capture individuals or businesses that have recently shown interest or have been newly entered as potential restaurant partners.
*   **Criteria:**
    *   `CreatedDate` within the last 24 hours (using a dynamic DateTime calculation for precision).
    *   `Status` indicating the lead is new and requires initial processing (e.g., 'Open - Not Contacted', 'New'). This avoids pulling leads created recently but already actioned or disqualified.
    *   The query implicitly targets unconverted leads by querying the `Lead` object itself.
*   **SOQL Query Example (using a bind variable for the DateTime):**
    ```soql
    /*
    In the calling application (Apex, Python, etc.), 'dateTime24HoursAgo'
    would be calculated as the current UTC date/time minus 24 hours.
    Example: 2024-03-14T10:30:00Z
    */
    SELECT
        Id,
        FirstName,
        LastName,
        Company,        // Typically the restaurant name
        Email,
        Phone,
        LeadSource,
        Status,         // To confirm it's a new/open lead
        Street,
        City,
        State,
        PostalCode,
        Country,
        CreatedDate,
        Description,    // For any initial notes or inquiry details
        Cuisine_Type_Inquiry__c, // Custom field for early details
        Estimated_Restaurant_Size_Inquiry__c // Custom field
    FROM
        Lead
    WHERE
        CreatedDate >= :dateTime24HoursAgo
        AND Status IN ('Open - Not Contacted', 'New', 'Pending Qualification') // Adjust status values as per org configuration
    ORDER BY
        CreatedDate DESC
    ```
*   **Output:** A list of `Lead` records including their IDs and relevant fields that provide initial information about the potential restaurant.

## 2. Query for New Accounts (Prospecting Stage)

*   **Object:** `Account`
*   **Purpose:** To identify restaurant entities that were created directly as Accounts (bypassing the Lead stage or immediately post-conversion) within the last 24 hours and are still considered new prospects.
*   **Criteria:**
    *   `CreatedDate` within the last 24 hours (using a dynamic DateTime calculation).
    *   A field indicating an early "Prospecting" or "New Signup" stage. This could be a standard field like `Type` or a custom field such as `Account.Lifecycle_Stage__c`, or a specific `RecordType`. For this example, we'll use a hypothetical custom field `Account.Prospecting_Status__c`.
*   **SOQL Query Example (using a bind variable for the DateTime):**
    ```soql
    /*
    'dateTime24HoursAgo' is calculated as in the Lead query.
    */
    SELECT
        Id,
        Name,           // Restaurant's official name
        Phone,
        Website,
        BillingStreet,  // Or ShippingStreet for physical location
        BillingCity,
        BillingState,
        BillingPostalCode,
        CreatedDate,
        Type,           // Standard field, could be 'Prospect'
        Prospecting_Status__c, // Hypothetical custom field, e.g., 'New Signup', 'Awaiting Outreach'
        Cuisine_Type__c,
        Restaurant_Size__c,
        Point_of_Sale_System__c,
        Current_Ordering_System__c,
        Primary_Contact__c, // Lookup to Contact
        (SELECT Id, FirstName, LastName, Email, Phone FROM Contacts WHERE Role_At_Restaurant__c = 'Owner' OR Role_At_Restaurant__c = 'Manager' LIMIT 1) // Subquery for key contact
    FROM
        Account
    WHERE
        CreatedDate >= :dateTime24HoursAgo
        AND (Prospecting_Status__c IN ('New Signup', 'Needs Initial Contact') OR Type = 'Prospect') // Adjust field/values as per org setup
        -- AND RecordType.Name = 'Restaurant Prospect' -- Alternative if using Record Types
    ORDER BY
        CreatedDate DESC
    ```
*   **Output:** A list of `Account` records representing newly created restaurants in an early prospecting stage, including their IDs and relevant operational details.

## 3. Combined Logic for AI Assistant Processing

The AI assistant will typically process these two lists to create a unified view of new sign-ups:

1.  **Execute Queries:** The AI will run both the Lead query and the Account query, using a dynamically calculated DateTime for the "last 24 hours" threshold.
2.  **Aggregate Results:** Collect the list of new Leads and new Accounts.
3.  **De-duplication & Prioritization:**
    *   A critical step is to avoid double-counting. If a Lead was created and then converted to an Account all within the 24-hour window, the AI should prioritize the Account record.
    *   This might involve checking if any identified Leads have a `ConvertedAccountId` that also appears in the new Accounts list, then choosing the Account record.
    *   If Leads are always converted before Accounts enter a "Prospecting" status relevant here, direct duplication might be less of an issue.
4.  **Unified List Creation:** The AI will form a single list of unique new sign-ups, potentially tagging each entry with its source (`Lead` or `Account`) and original ID.
5.  **Enrichment & Triage:** For each unique sign-up, the AI can then:
    *   Perform data validation and enrichment (e.g., check address validity, look for additional public information).
    *   Assess the completeness of provided information (e.g., cuisine type, contact details).
    *   Assign an initial priority score (e.g., populate `Account.AI_Assigned_Priority__c`).
    *   Determine and log the next best action (e.g., `Restaurant_Onboarding__c.Next_Step_AI__c` or create a Task for a sales representative).
    *   Initiate the creation of a `Restaurant_Onboarding__c` record if the sign-up meets predefined criteria for starting the onboarding process.

## 4. Dynamic Timeframe: "Last 24 Hours" Precision

*   **Using `LAST_N_DAYS:1` in SOQL:** This literal includes records from midnight yesterday up to the current time on the day the query is run (relative to the user's timezone or GMT depending on context). While simple, it's not a precise rolling 24-hour window. For example, if run at 1 PM, it includes all of yesterday and today up to 1 PM.
*   **Using Calculated DateTime (Recommended for True Rolling Window):**
    The application layer (Apex, Python, JavaScript, etc.) calling the SOQL should calculate the exact DateTime value for "24 hours ago from now" and use this in the `WHERE` clause (e.g., `CreatedDate >= :calculatedDateTime`).
    *   **Apex Example:** `DateTime twentyFourHoursAgo = System.now().addHours(-24);`
    *   **Python Example:** `twenty_four_hours_ago_dt = datetime.now(timezone.utc) - timedelta(hours=24)`
    *   **SOQL Format:** The DateTime should be formatted in ISO 8601 format (e.g., `YYYY-MM-DDTHH:MM:SSZ`) when constructing the query string if not using bind variables in Apex.

This precise calculation ensures that the "last 24 hours" is always a true rolling window from the moment of query execution, providing the most accurate and timely data for the AI assistant.The file `segmentation_logic_new_signups.md` has been created with the detailed SOQL queries and logic for identifying new restaurant sign-ups in the last 24 hours.

The document covers:
1.  **Query for New Leads:** Targets `Lead.CreatedDate` in the last 24 hours and specific `Status` values (e.g., 'Open - Not Contacted', 'New'), outputting relevant Lead fields.
2.  **Query for New Accounts (Prospecting Stage):** Targets `Account.CreatedDate` in the last 24 hours and a stage/status field (e.g., a hypothetical `Account.Prospecting_Status__c` or standard `Type`), outputting relevant Account fields including a subquery for key contacts.
3.  **Combined Logic:** Explains how the AI would execute these queries, aggregate results, handle potential de-duplication (prioritizing Accounts if a Lead was converted within the window), and then use the unified list for enrichment, triage, and determining next actions like populating `Account.AI_Assigned_Priority__c` or `Restaurant_Onboarding__c.Next_Step_AI__c`.
4.  **Dynamic Timeframe:** Details the use of `LAST_N_DAYS:1` for simplicity and the recommended, more precise method of calculating a specific DateTime (e.g., `System.now().addHours(-24)` in Apex or equivalent in other languages) for a true rolling 24-hour window.

This should fulfill the requirements of the subtask.
