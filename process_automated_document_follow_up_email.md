# Process: Automated Follow-Up for Pending Initial Documents

This document outlines the automated process for composing and sending a personalized follow-up email to restaurants that have not submitted their initial required documents within approximately 2 days of the welcome email or entry into the relevant onboarding stage.

## 1. Triggering Condition

This automated follow-up email process is typically initiated by a scheduled job (e.g., daily Apex batch job) that queries for `Restaurant_Onboarding__c` records meeting all the following criteria:

*   **Current Stage:** `Restaurant_Onboarding__c.Onboarding_Stage__c` is 'Initial Contact Made', 'Awaiting Documents', or a similar stage where document submission is the immediate expectation.
*   **Time Elapsed:**
    *   A relevant date field, such as `Restaurant_Onboarding__c.Initial_Welcome_Email_Sent_Date__c` OR `Restaurant_Onboarding__c.Stage_Entry_Date__c` (for the current "awaiting documents" stage), is older than 2 business days (e.g., `relevant_date_field <= System.today() - 2` - logic should account for business days if possible).
*   **Documents Not Yet Submitted:**
    *   A specific field indicating document submission, like `Restaurant_Onboarding__c.Documents_Submitted_Date__c`, is NULL.
    *   AND/OR the `Restaurant_Onboarding__c.Onboarding_Stage__c` has not progressed to a stage like 'Documents Submitted', 'Documents Submitted - Pending Verification', or 'Documents Verified'.
*   **Follow-Up Cap Not Reached:**
    *   A counter field, e.g., `Restaurant_Onboarding__c.Document_Follow_Up_Counter__c` (Number), is less than a predefined maximum (e.g., `< 2` or `< 3` to allow for a limited number of automated follow-ups for this specific issue).
*   **No Recent Similar Follow-up:**
    *   A date field, e.g., `Restaurant_Onboarding__c.Last_Document_Follow_Up_Sent_Date__c` (Date), is NULL or older than a specified interval (e.g., older than 2-3 business days) to prevent sending follow-ups too frequently.

## 2. Data Gathering from Salesforce

For each `Restaurant_Onboarding__c` record identified by the trigger, the system retrieves the following information:

*   **From `Restaurant_Onboarding__c`:**
    *   `Id`: ID of the onboarding record.
    *   `Account__c` (Lookup): The related `Account` record.
    *   `Onboarding_Owner__c` (Lookup): The `User` record of the assigned onboarding specialist.
    *   `Pending_Documents_List__c` (Text Area or Multi-select Picklist, optional): A field listing specific documents known to be pending (e.g., "Food License; Certificate of Insurance; Business Registration"). This field might be populated manually or by earlier AI analysis.
    *   `Document_Follow_Up_Counter__c` (Number): The current count of follow-ups sent for documents.

*   **From `Account` (via `Restaurant_Onboarding__c.Account__c`):**
    *   `Id`: Account ID.
    *   `Name`: The official name of the restaurant.
    *   `Primary_Contact__c` (Lookup): The main `Contact` for onboarding communications.
    *   `Document_Submission_Portal_Link__c` (URL): The link to the document submission portal.

*   **From `Contact` (via `Account.Primary_Contact__c`):**
    *   `Id`: Contact ID.
    *   `FirstName`: First name of the primary contact.
    *   `Email`: Email address of the primary contact.
    *   `HasOptedOutOfEmail` (Boolean): **Crucial check. If `TRUE`, this specific follow-up process for this contact must be aborted.**

*   **From `User` (Sender Data - via `Restaurant_Onboarding__c.Onboarding_Owner__c`):**
    *   `FirstName`: First name of the onboarding specialist to be used as the sender's apparent name.

## 3. LLM Prompt Design for Follow-Up Email

The system constructs a prompt to send to the LLM for generating the email content.

*   **Example LLM Prompt:**
    ```text
    You are an AI assistant for "SpeedyEats Platform". Your task is to draft a polite and helpful follow-up email to a restaurant that has not yet submitted their initial required documents after our welcome message.

    **Email Instructions:**
    *   **Tone:** Polite, helpful, understanding, and gently firm about the need for documents. Avoid sounding accusatory.
    *   **Purpose:** Serve as a gentle reminder, reiterate the importance of the documents for proceeding with their onboarding, and make it easy for them to submit.
    *   **Length:** Keep the email concise, approximately 2-3 short paragraphs.
    *   **Personalization:** Use the provided placeholders: `[Restaurant.Name]`, `[Contact.FirstName]`, `[Document_Submission_Portal_Link__c]`, `[Onboarding.Owner.FirstName]`.
    *   **Document Specificity (Crucial):**
        *   If `[Pending_Documents_List]` is provided and not empty (e.g., "Food License; Certificate of Insurance"), incorporate it naturally. Example: "We are particularly looking forward to receiving documents such as your [Pending_Document_1] and [Pending_Document_2] to move forward."
        *   If `[Pending_Documents_List]` is empty or not provided, use generic examples. Example: "These typically include items like your food license, certificate of insurance, and business registration details."
    *   **Key Action:** Encourage them to submit the documents via the `[Document_Submission_Portal_Link__c]`.

    **Email Structure Suggestion:**
    1.  **Subject Line:** Friendly reminder. Example: "Following Up: Your SpeedyEats Onboarding Documents for [Restaurant.Name]" or "A Gentle Reminder: Documents Needed for [Restaurant.Name] with SpeedyEats Platform"
    2.  **Salutation:** "Hi [Contact.FirstName],"
    3.  **Opening:** A polite check-in. Example: "Hope you're having a productive week! We're checking in on the onboarding process for [Restaurant.Name] with SpeedyEats Platform."
    4.  **Body Paragraph 1 (Reminder & Importance):** Gently remind them about the pending initial documents. Briefly reiterate that these documents are necessary for us to complete the setup of their profile and activate their services.
        *   (If specific documents known): "Our records show we're still awaiting a few key items, such as your [Pending_Document_1] and [Pending_Document_2], to complete this initial phase."
        *   (If generic): "This generally includes items like your food license, certificate of insurance, and business registration, which are essential for regulatory compliance and getting your account fully configured."
    5.  **Body Paragraph 2 (Call to Action & Support):** Re-share the `[Document_Submission_Portal_Link__c]` for easy access. Offer assistance: "If you have any questions about these documents or run into any issues with the submission, please don't hesitate to reach out. Your onboarding specialist, `[Onboarding.Owner.FirstName]`, is here to help."
    6.  **Closing:** Express anticipation for receiving the documents and moving to the next exciting stages of onboarding.
    7.  **Sign-off:** "Best regards," or "Thanks,"
    8.  **Sender Line:** "[Onboarding.Owner.FirstName] and the SpeedyEats Platform Team"

    **Data for this Email (to be injected by the system):**
    *   Restaurant.Name: "[Actual Restaurant Name from Account.Name]"
    *   Contact.FirstName: "[Actual Contact First Name from Contact.FirstName]"
    *   Document_Submission_Portal_Link__c: "[Actual Portal Link from Account.Document_Submission_Portal_Link__c]"
    *   Onboarding.Owner.FirstName: "[Actual Onboarding Owner First Name from User.FirstName]"
    *   Pending_Documents_List: "[Actual list from Restaurant_Onboarding__c.Pending_Documents_List__c, e.g., 'Food License;Certificate of Insurance' or leave empty if none specified]"

    Please generate ONLY the email subject and the full email body based on these instructions.
    ```

## 4. Salesforce Actions

*   **a. Pre-computation/Validation (Critical):**
    *   Verify `Contact.Email` is populated and valid.
    *   Verify `Contact.HasOptedOutOfEmail` is `FALSE`.
    *   If checks fail, abort sending to this contact and log an internal task for the `Onboarding_Owner__c` to investigate.

*   **b. Email Sending:**
    *   Use Apex `Messaging.SingleEmailMessage` as outlined in the welcome email process. Set appropriate sender (Org-Wide Email Address or Onboarding Owner, if configured) and recipient details.

*   **c. Activity Logging:**
    *   Create a `Task` record upon successful email dispatch.
    *   **Key Task Fields:**
        *   `WhoId`: `Contact.Id`.
        *   `WhatId`: `Account.Id`.
        *   `Restaurant_Onboarding_Task_Link__c` (Custom Lookup on Task to `Restaurant_Onboarding__c`): `Restaurant_Onboarding__c.Id`. (Highly recommended).
        *   `Subject`: The actual subject line sent (from LLM), e.g., "Email Sent: Follow-up on Initial Documents for [Restaurant.Name]".
        *   `Description`: Store the full body of the sent follow-up email.
        *   `Status`: 'Completed'.
        *   `Type`: 'Email - Follow-up' (Or a more specific type like 'Email - Document Follow-up').
        *   `ActivityDate`: `System.today()`.
        *   `OwnerId`: `Restaurant_Onboarding__c.Onboarding_Owner__c`.
        *   `Priority`: 'Normal'.

*   **d. Status Update on `Restaurant_Onboarding__c`:**
    *   `Last_Activity_Date_RO__c` (Custom Date/Time on `Restaurant_Onboarding__c`): Set to `System.now()`.
    *   `Last_Document_Follow_Up_Sent_Date__c` (Date on `Restaurant_Onboarding__c`): Set to `System.today()`.
    *   `Document_Follow_Up_Counter__c` (Number on `Restaurant_Onboarding__c`): Increment the existing value by 1.
    *   `Onboarding_Notes__c` (Long Text Area on `Restaurant_Onboarding__c`, optional): Append a system note, e.g., "Automated follow-up for pending documents sent. Follow-up count: [New Counter Value]."
    *   `Next_Step_AI__c` (Text Area on `Restaurant_Onboarding__c`): Could be updated to "Awaiting document submission from restaurant post follow-up email. Follow-up #[New Counter Value] sent."
    *   The main `Onboarding_Stage__c` typically does **not** change solely due to sending this follow-up email.

## 5. Error Handling & Frequency Capping

*   **LLM Interaction Failures:**
    *   **API Errors/Timeout:** Implement a retry mechanism (e.g., 2-3 retries).
    *   **Unusable Content:** If retries fail or content is invalid, **fallback to sending a predefined Salesforce Email Template** designed for document follow-up. This template should still use merge fields for essential personalization. Log the error internally (e.g., custom `AI_Interaction_Log__c` object or Task for admin) and notify the `Onboarding_Owner__c` or an AI admin that a standard template was used.
*   **Salesforce DML Errors (Task, Onboarding Update):**
    *   Use standard Apex try-catch blocks. Log exceptions. Implement a retry strategy for transient errors or escalate to an admin for persistent issues.
*   **Frequency Capping (Enforced by Triggering Logic):**
    *   The check against `Document_Follow_Up_Counter__c` in the triggering condition (Step 1) is the primary mechanism to prevent sending more than the allowed number of automated follow-ups for this specific purpose.
    *   The check against `Last_Document_Follow_Up_Sent_Date__c` ensures a minimum interval between follow-ups if the triggering job runs more frequently than the desired communication interval.

This process automates a crucial follow-up step, aiming to improve the efficiency of document collection and keep the onboarding process moving smoothly.Okay, the markdown document `process_automated_document_follow_up_email.md` has been created.

It includes:

1.  **Triggering Condition:**
    *   Specifies that a scheduled job identifies `Restaurant_Onboarding__c` records.
    *   Criteria: `Onboarding_Stage__c` ('Initial Contact Made', 'Awaiting Documents'), relevant date field (e.g., `Initial_Welcome_Email_Sent_Date__c` or `Stage_Entry_Date__c`) older than 2 business days, documents not yet submitted (e.g., `Documents_Submitted_Date__c` is NULL or stage hasn't progressed), `Document_Follow_Up_Counter__c` less than max, and `Last_Document_Follow_Up_Sent_Date__c` check for interval.

2.  **Data Gathering from Salesforce:**
    *   Lists fields from `Restaurant_Onboarding__c` (ID, Account lookup, Owner lookup, optional `Pending_Documents_List__c`, `Document_Follow_Up_Counter__c`).
    *   From `Account` (ID, Name, Primary Contact lookup, `Document_Submission_Portal_Link__c`).
    *   From `Contact` (ID, FirstName, Email, `HasOptedOutOfEmail`).
    *   From `User` (Onboarding Owner's FirstName).

3.  **LLM Prompt Design for Follow-Up Email:**
    *   Provides a detailed example prompt for "SpeedyEats Platform".
    *   Instructs LLM on tone (polite, helpful reminder), purpose (reiterate document importance), length, and personalization placeholders (`[Restaurant.Name]`, `[Contact.FirstName]`, `[Document_Submission_Portal_Link__c]`, `[Onboarding.Owner.FirstName]`).
    *   Includes logic for document specificity: Use `[Pending_Documents_List]` (e.g., "Food License;Certificate of Insurance") if available, otherwise use generic examples.
    *   Suggests email structure (Subject, Salutation, Opening, Body with reminder and importance, Call to Action with portal link and support offer, Closing, Sender Line).
    *   Includes a "Data for this Email" section showing where Salesforce data maps to placeholders.

4.  **Salesforce Actions:**
    *   **Pre-computation/Validation:** Critical check for `Contact.Email` validity and `Contact.HasOptedOutOfEmail = FALSE`. Abort and log internal task if issues.
    *   **Email Sending:** Via Apex `Messaging.SingleEmailMessage`.
    *   **Activity Logging:** Create a `Task` linked to `Contact`, `Account`, and `Restaurant_Onboarding__c` (via a custom lookup). Specifies key fields, including `Type` as 'Email - Follow-up'.
    *   **Status Update on `Restaurant_Onboarding__c`:** Update `Last_Activity_Date_RO__c`, `Last_Document_Follow_Up_Sent_Date__c`, increment `Document_Follow_Up_Counter__c`, optionally add to `Onboarding_Notes__c`, and update `Next_Step_AI__c`. Main stage usually doesn't change.

5.  **Error Handling & Frequency Capping:**
    *   **LLM Failures:** Retry, fallback to standard Salesforce Email Template, log error, notify admin/owner.
    *   **Salesforce DML Errors:** Standard try-catch, logging, retry/escalation.
    *   **Frequency Capping:** Enforced by `Document_Follow_Up_Counter__c` and `Last_Document_Follow_Up_Sent_Date__c` checks in the triggering logic.

This document should comprehensively guide the implementation of the automated document follow-up process.
