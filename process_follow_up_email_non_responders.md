# Process: Follow-Up Email to Non-Responders

This document outlines the process and Salesforce interactions for identifying contacts who haven't responded to an initial outreach email after approximately 2 days, and then composing and sending a follow-up email.

## Objective

To re-engage contacts who did not reply to a previous email, provide additional value or context, and accurately log the follow-up interaction in Salesforce.

## Prerequisites

1.  **Initial Email Logging:** First-touch emails are logged as `Task` records in Salesforce with identifiable subjects or markers.
2.  **Response Definition:** A clear definition of what constitutes a "response" is established (e.g., new inbound email, specific `CampaignMember.Status` change).
3.  **Follow-Up Templates:** Salesforce Email Templates designed for follow-up communication are available.

## Workflow Steps

### 1. Identify Non-Responding Contacts

*   **a. Query for Initial Outreach Tasks:**
    *   **Action:** Find `Task` records representing the initial emails sent approximately 2-10 days ago (this window is configurable).
    *   **Salesforce Interaction:** SOQL Query on `Task`.
    *   **SOQL Example:**
        ```soql
        SELECT Id, WhoId, WhatId, Subject, ActivityDate, OwnerId, (SELECT Id FROM Tasks WHERE Original_Email_Task__c = Task.Id LIMIT 1) /* If using custom field to link follow-ups */
        FROM Task
        WHERE Type = 'Email'
          AND (Subject LIKE '%First Touch - Introduction%' OR Campaign_Identifier__c = 'Q3Outreach') -- Use a reliable identifier
          AND ActivityDate >= :tenDaysAgo AND ActivityDate <= :twoDaysAgo -- Emails sent between 2 and 10 days ago
          AND Status IN ('Completed', 'Sent')
          -- AND Has_Follow_Up__c = FALSE -- Optional: Custom field on Task to track if a follow-up was already sent
        ```
        *   `:twoDaysAgo`: Calculated as `System.today().addDays(-2)` (or `System.now().addDays(-2)` for DateTime precision).
        *   `:tenDaysAgo`: Calculated as `System.today().addDays(-10)`.

*   **b. Determine "No Response" Status:**
    *   **Action:** For each initial `Task` (and its associated `Contact` - `Task.WhoId`), check for evidence of a response.
    *   **Salesforce Interactions (perform one or more as applicable):**
        1.  **Check for New Inbound Email Task:**
            ```soql
            SELECT Id FROM Task
            WHERE WhoId = :contactIdOfInitialEmail
              AND Type = 'Email'
              AND ActivityDate > :activityDateOfInitialEmail
              AND (NOT Subject LIKE '%Out of Office%') AND (NOT Subject LIKE '%Undeliverable%')
              -- AND Direction = 'Inbound' -- If available and reliable
            LIMIT 1
            ```
        2.  **Check CampaignMember Status (if initial email was part of a campaign):**
            ```soql
            SELECT Status FROM CampaignMember
            WHERE CampaignId = :campaignIdLinkedToInitialEmail
              AND ContactId = :contactIdOfInitialEmail
              AND Status IN ('Responded', 'Engaged', 'Meeting Scheduled') -- Define response statuses
            LIMIT 1
            ```
        3.  **Check Custom Interaction Objects (e.g., `Email_Interaction__oc`):**
            ```soql
            SELECT Id FROM Email_Interaction__oc
            WHERE Contact__c = :contactIdOfInitialEmail
              AND Response_Received_DateTime__c > :activityDateOfInitialEmail
            LIMIT 1
            ```
    *   **Logic:** If all relevant checks show no response, the contact is a candidate for follow-up.

### 2. Target Contact Validation (Re-check)

*   **Action:** Before sending the follow-up, ensure the contact is still valid for outreach.
*   **Salesforce Interaction:** SOQL Query on `Contact`.
*   **SOQL Example (for a `contactId` identified in Step 1):**
    ```soql
    SELECT Id, Email, HasOptedOutOfEmail, EmailBouncedDate
    FROM Contact
    WHERE Id = :contactId
      AND HasOptedOutOfEmail = FALSE
      AND EmailBouncedDate = NULL
    ```
    *   If this query returns no record, or if `HasOptedOutOfEmail` is true / `EmailBouncedDate` is set, do not send the follow-up.

### 3. Follow-Up Email Composition

*   **a. Select Follow-Up Email Template:**
    *   **Action:** Choose a Salesforce Email Template specifically designed for follow-ups.
    *   **Salesforce Interaction:** Query `EmailTemplate` or use a predefined template ID/name.
    *   **Selection Logic:** Template might be generic (e.g., "2-Day Follow-Up") or specific to the initial campaign/topic.
    *   **Example:** `EmailTemplate.DeveloperName = 'Follow_Up_No_Response_Generic_Day2'`

*   **b. Personalize Follow-Up Email Content:**
    *   **Action:** Merge Salesforce data and reference the previous communication.
    *   **Salesforce Interaction:** Use data from the `Contact` and `Account` records.
    *   **Content Strategy:**
        *   Reference the initial email: "I hope you're well. I'm following up on my email from [Date of original email / 'a couple of days ago'] regarding [brief original topic]."
        *   Offer new value, a different perspective, or a simpler call to action.
        *   Keep it concise.
    *   **AI Enhancement:** The AI can help summarize the previous email's purpose (from `Task.Description` or `Task.Subject`) and draft a contextually relevant follow-up.

### 4. Email Sending Mechanism

*   **Action:** Send the email using Salesforce capabilities.
*   **Methods:**
    *   **Apex (AI automation):** `Messaging.SingleEmailMessage`.
    *   **Salesforce Flow (Guided process):** Flow orchestrates the send.
    *   **Manual Send (User-driven, AI-assisted):** AI prepares details, user sends from UI.
*   **Opt-Out/Bounce Handling:** Step 2 validation and Salesforce's built-in mechanisms will handle this.

### 5. Logging the Follow-Up Interaction

*   **a. Create New Task (Activity) Record:**
    *   **Action:** Log the follow-up email as a new completed Task.
    *   **Salesforce Interaction:** Insert a `Task` record.
    *   **Key Task Fields:**
        *   `WhoId`: ID of the `Contact`.
        *   `WhatId`: ID of the related `Account`.
        *   `Subject`: E.g., "Email: Follow-Up - Re: [Original Topic] for {!Account.Name}".
        *   `Type`: 'Email'.
        *   `Status`: 'Completed' (or 'Sent').
        *   `ActivityDate`: Date follow-up was sent.
        *   `Description`: Full body of the sent follow-up email.
        *   `OwnerId`: Salesforce User ID responsible.
        *   `Original_Email_Task__c` (Custom Lookup(Task), optional): Link to the `Id` of the initial `Task` from Step 1.a. This is highly recommended for tracking conversation threads.

*   **b. Update Campaign Member Status (If Applicable):**
    *   **Action:** If part of a Salesforce Campaign, update the `CampaignMember.Status`.
    *   **Salesforce Interaction:** Update `CampaignMember` record.
    *   **Example Status:** 'Follow-up Sent - Day 2', 'Contacted - Attempt 2'.

*   **c. Mark Original Task (Optional but Recommended):**
    *   **Action:** Update the original `Task` (from Step 1.a) to indicate a follow-up has been sent.
    *   **Salesforce Interaction:** Update the original `Task` record.
    *   **Method:** Use a custom checkbox field like `Follow_Up_Sent__c = TRUE` on the `Task` object. This helps prevent sending multiple follow-ups for the same initial email if the identification query (Step 1.a) is run multiple times.

## Post-Follow-Up Monitoring (AI Considerations)

*   Continue to monitor for replies related to either the original or the follow-up email.
*   Analyze effectiveness: Which follow-up strategies or templates lead to better engagement?
*   Decision Point: If no response after this follow-up, determine next steps (e.g., different communication channel, pause outreach, close loop).

This process ensures that follow-up emails are timely, relevant, personalized, and meticulously tracked within Salesforce.
