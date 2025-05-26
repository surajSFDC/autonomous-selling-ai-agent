# Process: Automated Personalized Welcome Email to Newly Qualified Restaurant

This document outlines the automated process for composing and sending a personalized welcome email to a newly qualified restaurant. The process involves data retrieval from Salesforce, email content generation via a Large Language Model (LLM), and subsequent logging and status updates within Salesforce.

## 1. Triggering Condition

This automated welcome email process is initiated when:

*   **Primary Trigger:** A `Restaurant_Onboarding__c` record is created AND its `Onboarding_Stage__c` is set to 'New Signup / Pending Initiation' (or a similar initial stage like 'Awaiting Initial Contact', 'Information Gathering Kick-Off'). This indicates a restaurant has formally entered the onboarding pipeline.
*   **Pre-condition Check:** The system should verify that a welcome email has not already been sent for this specific `Restaurant_Onboarding__c.Id` to prevent duplicates. This can be done by checking a flag on the `Restaurant_Onboarding__c` record (e.g., `Welcome_Email_Sent_Flag__c` or `Initial_Welcome_Email_Sent_Date__c`).

## 2. Data Gathering from Salesforce

Before interacting with the LLM, the system must retrieve relevant data from Salesforce. The primary context is the `Restaurant_Onboarding__c` record that triggered the process.

*   **From `Restaurant_Onboarding__c`:**
    *   `Id`: ID of the onboarding record.
    *   `Account__c`: Lookup to the related `Account` record.
    *   `Onboarding_Owner__c`: Lookup to the `User` who is the onboarding specialist.

*   **From `Account` (via `Restaurant_Onboarding__c.Account__c`):**
    *   `Id`: For relating activities.
    *   `Name`: The official name of the restaurant (e.g., "The Gourmet Place").
    *   `Primary_Contact__c`: Lookup field to the main `Contact` record for the restaurant. This is crucial to identify the correct recipient.
    *   `Document_Submission_Portal_Link__c` (Custom URL field on Account or dynamically generated): A unique or generic link to the portal where the restaurant can submit initial required documents. *This field is assumed to exist or be generatable by the system.*

*   **From `Contact` (via `Account.Primary_Contact__c`):**
    *   `Id`: For addressing the email and logging activities.
    *   `FirstName`: The first name of the primary contact (e.g., "John").
    *   `Email`: The email address of the primary contact.
    *   `HasOptedOutOfEmail`: Standard Salesforce field. **If `TRUE`, the email must not be sent.**

*   **From `User` (Sender Data - via `Restaurant_Onboarding__c.Onboarding_Owner__c`):**
    *   `FirstName`: First name of the sender (e.g., "Sarah" - the onboarding specialist).
    *   `Email`: Email address of the sender (for From/Reply-To headers, if applicable and configured).
    *   `Signature`: Pre-formatted signature of the sender (if available and to be appended by the system post-LLM generation).

## 3. LLM Prompt Design for Welcome Email

The system will construct a prompt to send to the LLM for generating the email content.

*   **Example LLM Prompt:**
    ```text
    You are an AI assistant for "SpeedyEats Platform", a company that helps restaurants streamline their operations. Your task is to draft a personalized welcome email to a new restaurant that is beginning our onboarding process.

    **Email Instructions:**
    *   **Tone:** Friendly, welcoming, enthusiastic, and professional.
    *   **Purpose:** Welcome the restaurant to SpeedyEats Platform, express excitement for the partnership, briefly introduce the onboarding process, and clearly state the immediate next step (document submission).
    *   **Length:** Keep the email concise, ideally 3-4 short paragraphs.
    *   **Personalization:** Use the provided placeholders: `[Restaurant.Name]`, `[Contact.FirstName]`, `[Document_Submission_Portal_Link__c]`, `[Onboarding.Owner.FirstName]`.
    *   **Key Action:** Clearly guide them to submit their initial documents using the `[Document_Submission_Portal_Link__c]`. Mention this helps expedite their setup.

    **Email Structure Suggestion:**
    1.  **Subject Line:** Create an engaging subject line. Example: "Welcome to SpeedyEats Platform, [Restaurant.Name]! Let's Get You Started!"
    2.  **Salutation:** "Hi [Contact.FirstName],"
    3.  **Opening:** Welcome them to SpeedyEats Platform and express excitement about partnering with `[Restaurant.Name]`.
    4.  **Body Paragraph 1 (Onboarding Intro):** Briefly explain they're starting the onboarding journey. Mention that their dedicated onboarding specialist, `[Onboarding.Owner.FirstName]`, and our team are here to ensure a smooth setup.
    5.  **Body Paragraph 2 (Next Step - Documents):** State the immediate next step is to submit some initial documents. Provide the `[Document_Submission_Portal_Link__c]`. Briefly explain these documents are essential for getting their profile configured correctly and efficiently.
    6.  **Closing:** Reiterate excitement. Mention that `[Onboarding.Owner.FirstName]` will be in touch regarding next steps after document submission or to assist if they have questions.
    7.  **Sign-off:** "Best regards," or "Sincerely,"
    8.  **Sender Line:** "[Onboarding.Owner.FirstName] and the SpeedyEats Platform Team"

    **Data for this Email:**
    *   Restaurant.Name: "[Actual Restaurant Name from Salesforce Account.Name]"
    *   Contact.FirstName: "[Actual Contact First Name from Salesforce Contact.FirstName]"
    *   Document_Submission_Portal_Link__c: "[Actual Portal Link from Account.Document_Submission_Portal_Link__c]"
    *   Onboarding.Owner.FirstName: "[Actual Onboarding Owner First Name from Salesforce User.FirstName]"

    Please generate ONLY the email subject and the full email body based on these instructions.
    ```

## 4. Salesforce Actions

*   **a. Pre-computation/Validation (Critical):**
    *   Before any LLM call or email sending, the system MUST check:
        *   Is `Contact.Email` populated and valid?
        *   Is `Contact.HasOptedOutOfEmail` set to `FALSE`?
    *   If email is missing/invalid or contact has opted out, the process for this recipient must be aborted. An internal task should be created for the `Onboarding_Owner__c` to manually review or obtain correct contact details.

*   **b. Email Sending (Post LLM Generation):**
    *   **Method:** Use Apex `Messaging.SingleEmailMessage`.
    *   **Recipient:** Set `setTargetObjectId(contactId)` or `setToAddresses(new List<String>{contactEmail})`.
    *   **Sender:**
        *   `setOrgWideEmailAddressId()`: Use a pre-configured Organization-Wide Email Address.
        *   Or, if permissions allow, send on behalf of the `Onboarding_Owner__c` by setting `setSenderDisplayName()` and ensuring Reply-To is correctly set.
    *   **Subject & Body:** Use the subject and HTML body generated by the LLM.
    *   `setHtmlBody(llmGeneratedHtmlBody)`
    *   `setSubject(llmGeneratedSubject)`
    *   A plain text version (`setPlainTextBody()`) should also be included.

*   **c. Activity Logging:**
    *   **Action:** Create a `Task` record upon successful email dispatch.
    *   **Key Task Fields:**
        *   `WhoId`: `Contact.Id` (the recipient).
        *   `WhatId`: `Account.Id` (the restaurant).
        *   `Restaurant_Onboarding_Task_Link__c` (Custom Lookup on Task to `Restaurant_Onboarding__c`): `Restaurant_Onboarding__c.Id` (This direct link is highly recommended for focused activity tracking on the onboarding record).
        *   `Subject`: The actual subject line sent (from LLM). E.g., "Email Sent: Welcome to SpeedyEats Platform, [Restaurant.Name]!"
        *   `Description`: Store the full body of the email sent (from LLM).
        *   `Status`: 'Completed'.
        *   `Type`: 'Email'.
        *   `ActivityDate`: `System.today()`.
        *   `OwnerId`: `Restaurant_Onboarding__c.Onboarding_Owner__c`.
        *   `Priority`: 'Normal'.

*   **d. Status Update on `Restaurant_Onboarding__c`:**
    *   **Action:** Update the onboarding record.
    *   **Fields to Update:**
        *   `Onboarding_Stage__c`: Set to 'Initial Contact Made' (or a more suitable subsequent stage like 'Awaiting Documents').
        *   `Last_Activity_Date_RO__c` (Custom Date/Time field on `Restaurant_Onboarding__c`): Set to `System.now()`.
        *   `Initial_Welcome_Email_Sent_Date__c` (Custom Date field on `Restaurant_Onboarding__c`): Set to `System.today()`. This also serves as the flag to prevent re-sending.
        *   `Next_Step_AI__c`: Could be updated to "Awaiting document submission from restaurant."

## 5. Error Handling

*   **LLM Interaction Failures:**
    *   **API Errors (Timeout, Service Unavailable):** Implement a retry mechanism (e.g., 2-3 retries with delays).
    *   **Unusable Content (Empty, Gibberish, Failed Validation):**
        1.  **Fallback to Standard Template:** If LLM fails after retries, the system MUST automatically send a predefined Salesforce Email Template. This template should still use merge fields for basic personalization (`[Restaurant.Name]`, `[Contact.FirstName]`, `[Document_Submission_Portal_Link__c]`).
        2.  **Log Error:** Create an internal error log (e.g., custom object `AI_Interaction_Log__c` or a Task for an AI admin).
        3.  **Notify Owner/Admin:** Alert the `Onboarding_Owner__c` or a system admin that personalized email generation failed and a standard template was used.
*   **Salesforce DML Errors (Task Creation, Onboarding Update):**
    *   Use standard Apex try-catch blocks.
    *   Log any DML exceptions.
    *   Implement a retry strategy if appropriate for transient errors, or escalate to an admin for persistent issues. Data consistency (e.g., email sent but task not logged) should be monitored.

This process ensures a personalized and automated welcome for new restaurants, efficiently leveraging LLM capabilities while maintaining robust error handling and Salesforce integration.Okay, the markdown document `process_automated_welcome_email.md` has been created.

It covers:

1.  **Triggering Condition:**
    *   Primary: `Restaurant_Onboarding__c` record created with `Onboarding_Stage__c` = 'New Signup / Pending Initiation' (or similar).
    *   Pre-condition: Check a flag or date field on `Restaurant_Onboarding__c` to prevent duplicate welcome emails.

2.  **Data Gathering from Salesforce:**
    *   Lists key fields from `Restaurant_Onboarding__c` (as the context), `Account` (e.g., `Name`, `Primary_Contact__c`, `Document_Submission_Portal_Link__c`), `Contact` (e.g., `FirstName`, `Email`, `HasOptedOutOfEmail`), and `User` (for sender details like `Onboarding_Owner__c`'s first name).

3.  **LLM Prompt Design for Welcome Email:**
    *   Provides a structured example prompt for the LLM.
    *   Instructs the LLM on tone (friendly, welcoming), purpose (start onboarding, outline next steps), length, personalization placeholders (`[Restaurant.Name]`, `[Contact.FirstName]`, `[Document_Submission_Portal_Link__c]`, `[Onboarding.Owner.FirstName]`), and desired email structure (Subject, Salutation, Opening, Body paragraphs for intro and document submission, Closing, Sender line).
    *   Includes a "Data for this Email" section with placeholders for the actual data to be injected.

4.  **Salesforce Actions:**
    *   **Pre-computation/Validation:** Emphasizes checking `Contact.Email` validity and `Contact.HasOptedOutOfEmail` status *before* LLM call or sending.
    *   **Email Sending:** Recommends `Messaging.SingleEmailMessage`, setting recipient, sender (Org-Wide Email Address or Onboarding Owner), and using LLM-generated subject and HTML/Plain Text body.
    *   **Activity Logging:** Details creating a `Task` record linked to `Contact`, `Account`, and ideally directly to `Restaurant_Onboarding__c` (via a custom lookup). Specifies key Task fields like `WhoId`, `WhatId`, custom lookup, `Subject`, `Description` (email body), `Status`, `Type`, `ActivityDate`, `OwnerId`.
    *   **Status Update:** Describes updating `Restaurant_Onboarding__c.Onboarding_Stage__c` (e.g., to 'Initial Contact Made'), `Last_Activity_Date_RO__c`, `Initial_Welcome_Email_Sent_Date__c`, and potentially `Next_Step_AI__c`.

5.  **Error Handling:**
    *   **LLM Interaction Failures:** Covers API errors (retries) and unusable content, with a fallback to a standard Salesforce Email Template and notification to admin/owner.
    *   **Salesforce DML Errors:** Mentions standard try-catch blocks, error logging, and potential retry or admin escalation.

This document should provide a comprehensive guide for implementing the automated welcome email process.
