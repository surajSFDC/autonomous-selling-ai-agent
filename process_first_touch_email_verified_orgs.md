# Process: First-Touch Email to Verified Organizations

This document outlines the process and Salesforce interactions for composing and sending a first-touch email to contacts within organizations that have been identified as "verified" (i.e., `Account.Verified_Email_Organization__c = TRUE`).

## Objective

To effectively introduce the company's service to relevant contacts in verified organizations, ensuring the interaction is personalized, respects recipient preferences, and is accurately logged in Salesforce.

## Prerequisites

1.  **Accurate Verification Data:** The custom field `Account.Verified_Email_Organization__c` (Boolean) is reliably populated and maintained.
2.  **Email Templates:** Relevant Salesforce Email Templates exist, designed for first-touch outreach and incorporating merge fields.
3.  **Contact Prioritization Logic:** A method exists to identify suitable target contacts (e.g., a `Primary_Contact__c` field or use of `Decision_Maker_Status__c` on Contact).

## Workflow Steps

### 1. Target Selection

*   **a. Identify Target Accounts:**
    *   **Action:** Query Salesforce for Accounts where `Verified_Email_Organization__c = TRUE`.
    *   **Salesforce Interaction:** SOQL Query.
    *   **SOQL Example:**
        ```soql
        SELECT Id, Name, Industry -- other fields for context
        FROM Account
        WHERE Verified_Email_Organization__c = TRUE
          -- AND Id NOT IN (SELECT AccountId FROM CampaignMember WHERE Campaign.Name = 'Recent Outreach Campaign') -- Optional: Exclude recently contacted
          -- AND LastActivityDate < LAST_N_DAYS:30 -- Optional: Exclude recently active
        ```
    *   **Considerations:** The AI can apply further filters (e.g., industry, size, not part of an existing recent campaign) to refine the list of target Accounts.

*   **b. Identify Target Contacts within Accounts:**
    *   **Action:** For each selected Account, identify one or more suitable Contacts.
    *   **Salesforce Interaction:** SOQL Query against the Contact object.
    *   **Contact Prioritization Rules (example sequence):**
        1.  Contact marked as `Primary_Contact__c = TRUE`.
        2.  Contact with `Decision_Maker_Status__c` of 'Decision Maker' or 'Influencer'.
        3.  Most recently modified Contact not matching opt-out criteria.
    *   **Crucial Filtering:** Always exclude Contacts where `HasOptedOutOfEmail = TRUE` or `EmailBouncedDate` is not null.
    *   **SOQL Example (for a given `accountId`):**
        ```soql
        SELECT Id, FirstName, LastName, Email, Title, HasOptedOutOfEmail, EmailBouncedDate
        FROM Contact
        WHERE AccountId = :accountId
          AND (Primary_Contact__c = TRUE OR Decision_Maker_Status__c IN ('Decision Maker', 'Influencer')) -- Adjust logic as needed
          AND HasOptedOutOfEmail = FALSE
          AND EmailBouncedDate = NULL
        ORDER BY LastModifiedDate DESC -- Or other relevant ordering
        LIMIT 1 -- Or more if multiple contacts are targeted per Account
        ```

### 2. Email Composition

*   **a. Select Email Template:**
    *   **Action:** Choose an appropriate Salesforce Email Template.
    *   **Salesforce Interaction:** Query `EmailTemplate` object or use a predefined template ID/name.
    *   **Selection Logic:**
        *   Based on campaign goals (e.g., "First Touch Introduction").
        *   Potentially tailored by Account Industry or other attributes if multiple template versions exist.
        *   The AI could have a default template or use rules to select one.
    *   **Example:** `EmailTemplate.DeveloperName = 'Verified_Org_Initial_Outreach_v1'`

*   **b. Personalize Email Content:**
    *   **Action:** Merge Salesforce data into the selected template.
    *   **Salesforce Interaction:** Data from the target `Contact` and `Account` records (retrieved in Step 1) is used to populate merge fields in the template.
    *   **Common Merge Fields:**
        *   `{!Contact.FirstName}`
        *   `{!Account.Name}`
        *   `{!User.FirstName}` (Sender's first name)
        *   `{!User.Email}` (Sender's email)
    *   **AI Enhancement (Optional):** The AI may suggest further textual customizations beyond standard merge fields, based on research or specific Account/Contact details to make the email more impactful.

### 3. Email Sending Mechanism

*   **a. Utilize Salesforce Email Sending Capabilities:**
    *   **Method 1: Apex (for automation by AI):**
        *   Use `Messaging.SingleEmailMessage` or `Messaging.MassEmailMessage`.
        *   Set target object ID (`setTargetObjectId(contactId)`), template ID (`setTemplateId(templateId)`), and potentially `setWhatId(accountId)` for linking activity to the account too.
    *   **Method 2: Salesforce Flow (for guided process):**
        *   A Flow could orchestrate the selection and sending process, invoked by the AI or a user.
    *   **Method 3: Manual Send (user-driven, AI-assisted):**
        *   The AI prepares the email details (recipient, template, perhaps a draft body), and the user sends it via the Salesforce UI (e.g., from the Contact's Activity Timeline).

*   **b. Adherence to Opt-Outs and Bounce Management:**
    *   **Built-in Salesforce Behavior:**
        *   `HasOptedOutOfEmail`: Salesforce automatically prevents sending to Contacts with this flag checked when using standard send methods. The query in Step 1.b already filters these out.
        *   `EmailBouncedDate` / `EmailBouncedReason`: Salesforce populates these fields if an email hard bounces. The query in Step 1.b also filters these out.
    *   **Best Practice:** Always ensure queries for target contacts explicitly filter out opted-out and bounced emails as a primary step.

### 4. Logging the Interaction

*   **a. Create a Task (Activity) Record:**
    *   **Action:** Log the email send as a completed Task.
    *   **Salesforce Interaction:** Insert a `Task` record.
    *   **Key Task Fields to Populate:**
        *   `WhoId`: ID of the `Contact` (recipient).
        *   `WhatId`: ID of the related `Account`.
        *   `Subject`: E.g., "Email: First Touch - Introduction to [Service/Product] for {!Account.Name}".
        *   `Type`: 'Email'.
        *   `Status`: 'Completed' (or a custom status like 'Sent').
        *   `ActivityDate`: Date email was sent (usually `System.today()`).
        *   `Description`: Full body of the sent email for historical reference.
        *   `OwnerId`: The Salesforce User ID responsible for the outreach (e.g., the user the AI is acting on behalf of, or a generic integration user).
        *   `Priority`: 'Normal'.

*   **b. Update Campaign Member Status (If Applicable):**
    *   **Action:** If the outreach is part of a Salesforce Campaign, update the Contact's status in that campaign.
    *   **Salesforce Interaction:**
        1.  Ensure the Contact is a `CampaignMember` of the relevant Campaign. If not, add them.
        2.  Update `CampaignMember.Status` to a value like 'Email Sent - First Touch' or 'Responded'.
    *   **Example (Apex Conceptual):**
        ```apex
        CampaignMember cm = [SELECT Id FROM CampaignMember WHERE ContactId = :contactId AND CampaignId = :campaignId LIMIT 1];
        if (cm != null) {
            cm.Status = 'Sent - AI First Touch';
            update cm;
        } else {
            CampaignMember newCm = new CampaignMember(
                ContactId = contactId,
                CampaignId = campaignId,
                Status = 'Sent - AI First Touch'
            );
            insert newCm;
        }
        ```

## Post-Sending Monitoring (AI Considerations)

*   **Reply Tracking:** The AI could monitor for incoming emails from the contact (new `Task` or `EmailMessage` records related to the `Contact`).
*   **Engagement Tracking:** If using tracked links or specific Salesforce features (like High Velocity Sales or Marketing Cloud integrations), monitor email opens and clicks.
*   **Automated Follow-up:** Based on lack of response or specific engagement signals, the AI could schedule or suggest follow-up actions.

This structured process ensures that first-touch emails are sent effectively, are well-documented in Salesforce, and align with best practices for customer communication.
