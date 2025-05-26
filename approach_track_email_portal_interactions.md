# Approach: Track Email Opens and Document Portal Interactions

This document outlines the conceptual approach for tracking email open events and document submission portal interactions within Salesforce. This tracking is crucial for understanding engagement and automating responses in the restaurant onboarding process.

## 1. Email Open Tracking (Conceptual Salesforce Logging)

*   **Acknowledgement of Limitation:** Standard Salesforce Apex email sending (e.g., `Messaging.SingleEmailMessage`) does **not** provide out-of-the-box email open tracking.
*   **Dependency on External System:** This functionality relies on an external email sending service (e.g., SendGrid, Mailgun, Amazon SES, or marketing automation platforms like Marketing Cloud or Pardot) that can:
    1.  Embed tracking pixels in emails.
    2.  Detect when these pixels are loaded (email open).
    3.  Send this open event data back to Salesforce via an API call (e.g., a webhook).

*   **Salesforce Logging Options for 'Open' Events:**

    *   **Option A: On the `Task` Record (Associated with the Sent Email):**
        *   **Field 1:** `Task.Email_Opened_DateTime__c` (Custom DateTime field): Stores the timestamp of the first open.
        *   **Field 2:** `Task.Email_Open_Count__c` (Custom Number field, optional): Stores the number of times the email was opened, if the external system provides this.
        *   **Pros:** Directly links the open event to the specific email activity. Easy to see if *that particular email* was opened.
        *   **Cons:** Requires identifying the correct Task record in Salesforce when the webhook is received (e.g., using a unique identifier passed to the email service and included in the webhook). Reporting across multiple opens for a single contact/onboarding might require more complex report grouping.

    *   **Option B: On the `Restaurant_Onboarding__c` Record:**
        *   **Field 1:** `Restaurant_Onboarding__c.Last_Email_Open_Date__c` (Custom DateTime field): Stores the timestamp of the most recent email open related to this onboarding.
        *   **Field 2:** `Restaurant_Onboarding__c.Any_Email_Opened_Flag__c` (Custom Boolean field, optional): A flag that is set to TRUE if any email sent during this onboarding has been opened.
        *   **Pros:** Centralizes key engagement signals on the main onboarding record.
        *   **Cons:** Loses specificity of *which* email was opened unless additional logging (like creating a child "Interaction" record) is implemented.

    *   **Option C: Dedicated `Email_Interaction__oc` Custom Object (Recommended for Granularity):**
        *   This object (as previously proposed in `salesforce_object_outline.md`) would have fields like:
            *   `Contact__c` (Lookup to Contact)
            *   `Account__c` (Lookup to Account)
            *   `Restaurant_Onboarding__c` (Lookup to Restaurant_Onboarding__c)
            *   `Related_Email_Task__c` (Lookup to the original Task for the sent email)
            *   `Interaction_Type__c` (Picklist: 'Sent', 'Opened', 'Clicked')
            *   `Interaction_DateTime__c` (DateTime)
            *   `Email_Subject_Line__c` (Text, copied from Task for context)
        *   **Pros:** Provides the most detailed and flexible tracking. Each open is a distinct record. Allows for tracking multiple opens of the same email or opens of different emails related to the same onboarding. Simplifies reporting on open trends.
        *   **Cons:** Higher data volume. Requires creation of a new record for each open event.

## 2. Document Submission Portal Click/Submission Tracking

*   **Assumption:** The external document submission portal is capable of making an API call back to Salesforce (e.g., using Salesforce REST API with OAuth 2.0 authentication) when specific events occur, like a link click or a document submission.
*   **Salesforce Fields to Update:**

    *   **Upon Portal Link Click (If Tracked and Valuable):**
        *   **Target Object:** `Restaurant_Onboarding__c`
        *   **Field:** `Restaurant_Onboarding__c.Portal_Link_Last_Clicked_Date__c` (Custom DateTime field)
        *   **Logic:** The portal, when a user clicks a link that leads to it (e.g., from an email), could make an API call to Salesforce to update this field with the current timestamp. This indicates interest/intent even if documents aren't submitted immediately.
        *   **Note:** This is less common than tracking actual submissions due to potential for accidental clicks or multiple clicks.

    *   **Upon Successful Document Submission:**
        *   **Target Object:** `Restaurant_Onboarding__c`
        *   **Key Fields to Update:**
            *   `Restaurant_Onboarding__c.Documents_Submitted_Date__c` (Custom DateTime field): Set to the timestamp of the successful submission.
            *   `Restaurant_Onboarding__c.Onboarding_Stage__c` (Picklist): Automatically update to 'Documents Submitted' or 'Documents Submitted - Pending Verification'. This is a critical automation step.
            *   **(Optional) `Restaurant_Onboarding__c.Submitted_Documents_List__c` (Text Area or Multi-select Picklist):** If the portal API can send a list or manifest of *which* documents were submitted (e.g., "Food License", "Insurance Certificate"), this field can be updated. This helps in verifying completeness.
            *   **(Optional) `Restaurant_Onboarding__c.Pending_Documents_List__c` (Text Area or Multi-select Picklist):** If the portal knows which documents were submitted, this field could be updated by removing the submitted items from the pending list. This requires more sophisticated API interaction.
        *   **Additional Actions:**
            *   Create a `Task` for the `Onboarding_Owner__c` to review the submitted documents. Subject: "Action Required: Review Submitted Documents for [Restaurant Name]".
            *   Send an automated acknowledgement email to the restaurant contact (though this might be handled by the portal itself).

## 3. Data Points for Analytics

The tracked data points from email opens and portal interactions are valuable for analytics:

*   **Email Engagement:**
    *   Open rates (if tracked) for specific emails or email templates can indicate effectiveness of subject lines or relevance of content.
    *   Timing of opens can show when contacts are most active.
*   **Funnel Conversion/Drop-off:**
    *   Track conversion from "Email Sent" -> "Email Opened" -> "Portal Link Clicked" (if tracked) -> "Documents Submitted".
    *   Identify where in this initial funnel contacts are dropping off.
*   **Time-to-Action:**
    *   Calculate time from `Initial_Welcome_Email_Sent_Date__c` to `Portal_Link_Last_Clicked_Date__c` or `Documents_Submitted_Date__c`.
    *   Measure how long restaurants take to respond to document requests.
*   **Effectiveness of Follow-Ups:**
    *   Compare open/submission rates after follow-up emails versus initial emails.
*   **Onboarding Velocity:**
    *   `Documents_Submitted_Date__c` is a key milestone contributing to overall onboarding duration.

## 4. AI Assistant's Use of Tracking Data

The AI assistant can leverage this tracking data to enhance its intelligence and proactiveness:

*   **Adjust Follow-Up Timing/Content:**
    *   If an email (e.g., welcome email or document request) is opened multiple times by the contact (`Task.Email_Open_Count__c` or multiple `Email_Interaction__oc` records) but the `Portal_Link_Last_Clicked_Date__c` or `Documents_Submitted_Date__c` on `Restaurant_Onboarding__c` is not updated, the AI could:
        *   Trigger a slightly delayed or different follow-up message, acknowledging potential interest but highlighting the next step.
        *   Suggest to the `Onboarding_Owner__c` that the contact might be interested but facing a hurdle.
    *   If an email remains unopened after a certain period, the AI could suggest trying a different contact at the restaurant or a different communication channel.
*   **Flag for Human Attention:**
    *   "High Engagement, No Action": If an email is opened many times, or the portal link is clicked multiple times, but documents are not submitted within a reasonable timeframe, the AI can flag the `Restaurant_Onboarding__c` record (e.g., update `Onboarding_Status__c` to 'Needs Attention' or create a high-priority Task for the `Onboarding_Owner__c`) for manual intervention. The contact might be confused or facing technical difficulties.
*   **Predictive Onboarding Path:**
    *   Over time, by analyzing patterns of opens, clicks, and submission times across many onboarding instances, the AI could start to predict which restaurants are more likely to complete steps quickly versus those that might need more proactive support.
*   **Personalize Next Communication:**
    *   If the AI knows a contact has opened a specific email but not clicked the portal link, its next interaction (e.g., a summary for the onboarding owner, or a subsequent automated email) could be tailored to specifically address potential barriers to clicking the link or to re-emphasize the portal's purpose.
*   **Confirm Engagement before Critical Steps:**
    *   Before triggering resource-intensive internal processes, the AI could check for positive engagement signals (like email opens or portal clicks) as a prerequisite.

By effectively logging these interaction events in Salesforce, both human users and the AI assistant gain valuable insights to optimize the restaurant onboarding experience.Okay, the markdown document `approach_track_email_portal_interactions.md` has been created.

It covers:

1.  **Email Open Tracking (Conceptual Salesforce Logging):**
    *   Acknowledges that standard Apex email sends lack this feature and that it relies on an external email sending service with tracking pixel capabilities and webhook/API callbacks.
    *   Discusses three Salesforce logging options for 'open' events:
        *   **Option A: On the `Task` Record:** Using custom fields like `Task.Email_Opened_DateTime__c` or `Task.Email_Open_Count__c`. Pros: direct link to email activity. Cons: identifying the correct Task via webhook.
        *   **Option B: On `Restaurant_Onboarding__c`:** Using fields like `Restaurant_Onboarding__c.Last_Email_Open_Date__c`. Pros: centralizes signals. Cons: loses specificity of which email.
        *   **Option C: Dedicated `Email_Interaction__oc` Custom Object (Recommended for Granularity):** Details fields like `Contact__c`, `Restaurant_Onboarding__c`, `Related_Email_Task__c`, `Interaction_Type__c` ('Opened'), `Interaction_DateTime__c`. Pros: detailed, flexible tracking. Cons: higher data volume.

2.  **Document Submission Portal Click/Submission Tracking (Conceptual Salesforce Logging):**
    *   Assumes the external portal can make API calls back to Salesforce.
    *   **Upon Portal Link Click (If Tracked):** Updates `Restaurant_Onboarding__c.Portal_Link_Last_Clicked_Date__c`.
    *   **Upon Successful Document Submission:**
        *   Updates `Restaurant_Onboarding__c.Documents_Submitted_Date__c`.
        *   Updates `Restaurant_Onboarding__c.Onboarding_Stage__c` (e.g., to 'Documents Submitted').
        *   Optionally updates `Restaurant_Onboarding__c.Submitted_Documents_List__c` or `Pending_Documents_List__c`.
        *   Suggests creating a Task for document review and potentially sending an acknowledgement.

3.  **Data Points for Analytics:**
    *   Mentions how tracked data (open dates, click dates, submission dates) feeds into analytics for email engagement (open rates), funnel conversion/drop-off, time-to-action, effectiveness of follow-ups, and overall onboarding velocity.

4.  **AI Assistant's Use of Tracking Data:**
    *   **Adjust Follow-Up:** If email opened multiple times but no portal click/submission, AI could trigger different follow-up or alert owner. If unopened, suggest different contact/channel.
    *   **Flag for Human Attention:** "High Engagement, No Action" (many opens/clicks, no submission) can flag `Restaurant_Onboarding__c` for manual intervention.
    *   **Predictive Onboarding Path:** AI could learn patterns to predict onboarding speed.
    *   **Personalize Next Communication:** Tailor next steps based on known engagement.
    *   **Confirm Engagement:** Check for signals before triggering resource-intensive internal steps.

This document should provide a solid conceptual framework for the tracking mechanisms.
