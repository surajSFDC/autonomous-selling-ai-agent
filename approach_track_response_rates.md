# Approach: Track and Log Response Rates for Outreach Campaigns

This document defines the strategy for tracking, calculating, and logging response rates and other key performance indicators for email outreach campaigns, utilizing Salesforce as the primary data source.

## 1. Key Metrics

The following metrics are crucial for understanding campaign performance:

*   **a. Email Open Rate (Conditional):**
    *   **Definition:** Percentage of unique contacts who opened a campaign email out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Opens / Number of Emails Delivered) * 100`
    *   **Dependency:** Requires an email system integrated with Salesforce that tracks opens (e.g., Marketing Cloud, Pardot, Sales Cloud Einstein Activity Capture with certain configurations, third-party email services). Standard Apex email sends do not track opens.

*   **b. Email Click-Through Rate (CTR) (Conditional):**
    *   **Definition:** Percentage of unique contacts who clicked a link in a campaign email out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Clicks / Number of Emails Delivered) * 100`
    *   **Dependency:** Similar to Open Rate, relies on tracking capabilities of the sending platform.

*   **c. Email Reply Rate (Core Metric):**
    *   **Definition:** Percentage of unique contacts who replied to a campaign email out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Contacts Replied / Number of Emails Delivered) * 100`

*   **d. Positive Reply Rate:**
    *   **Definition:** Percentage of unique contacts who provided a positive reply (e.g., showed interest, requested a meeting) out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Contacts with Positive Reply / Number of Emails Delivered) * 100`
    *   **Note:** Requires reply content analysis (manual or AI-driven).

*   **e. Negative Reply Rate (Includes Opt-Outs via Reply):**
    *   **Definition:** Percentage of unique contacts who provided a negative reply (e.g., not interested, request to unsubscribe) out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Contacts with Negative Reply / Number of Emails Delivered) * 100`

*   **f. Conversion Rate (Campaign-Specific Goal):**
    *   **Definition:** Percentage of unique contacts who completed a specific desired action (e.g., demo scheduled, trial signup) as a result of the campaign, out of successfully delivered emails.
    *   **Formula:** `(Number of Unique Contacts Converted / Number of Emails Delivered) * 100`

*   **g. Bounce Rate:**
    *   **Definition:** Percentage of emails that failed to deliver (hard or soft bounces) out of the total emails sent.
    *   **Formula:** `(Number of Bounced Emails / Total Emails Sent) * 100`

*   **h. Delivery Rate:**
    *   **Definition:** Percentage of emails that were successfully delivered.
    *   **Formula:** `(Total Emails Sent - Number of Bounced Emails) / Total Emails Sent * 100`
    *   **Note:** "Number of Emails Delivered" used in other rate calculations is `Total Emails Sent - Number of Bounced Emails`.

## 2. Data Sources in Salesforce

*   **a. `Campaign` and `CampaignMember` Objects:**
    *   **`Campaign`:**
        *   Central record for an outreach initiative.
        *   Standard fields: `NumberOfSentEmails`, `NumberOfResponses` (can be driven by `CampaignMember.HasResponded`).
        *   Custom fields: To store calculated rates (e.g., `Actual_Reply_Rate__c`, `Positive_Reply_Rate__c`, `Campaign_Conversion_Rate__c`).
    *   **`CampaignMember`:**
        *   Tracks each recipient (`ContactId`, `LeadId`) within a `Campaign`.
        *   `Status` (Picklist): Critical for tracking. Values should be well-defined, e.g.:
            *   'Targeted', 'Scheduled', 'Sent', 'Delivered'
            *   'Opened', 'Clicked' (if tracking is available)
            *   'Responded - General', 'Responded - Positive', 'Responded - Negative', 'Responded - Neutral', 'Responded - Out of Office'
            *   'Converted - Demo Scheduled', 'Converted - Trial Started'
            *   'Bounced - Hard', 'Bounced - Soft'
            *   'Unsubscribed - Via Reply', 'Unsubscribed - Via Link'
        *   `HasResponded` (Standard Boolean): Automatically true if `CampaignMember.Status` is configured as a "Responded" type.
        *   Custom fields: `Responded_DateTime__c`, `Opened_DateTime__c`, `Sentiment_Category__c`.

*   **b. `Task` Records (Email Activities):**
    *   **Identifying Sent Emails:** `Type` = 'Email', `Subject` may contain campaign identifiers, `ActivityDate` = send date, `WhoId` = recipient, `Status` = 'Completed'/'Sent'.
    *   **Identifying Replies:** Requires logic to find subsequent inbound `Task` records from the same `WhoId` after the outreach Task's `ActivityDate`. The AI must parse `Task.Description` (email body) for sentiment and intent, and to distinguish actual replies from auto-responders. This is complex and often requires specific keywords or AI analysis.
    *   **Linking:** The AI or automation would then update the `CampaignMember.Status` or related custom fields based on this analysis.

*   **c. Custom Object: `Email_Interaction__oc` (If Implemented):**
    *   As outlined in `salesforce_object_outline.md`, fields like `Contact__c`, `Campaign__c`, `Sent_DateTime__c`, `Opened_DateTime__c`, `Clicked_DateTime__c`, `Response_Received_DateTime__c`, `Response_Sentiment__c` provide a structured way to log these events.
    *   **Benefit:** Greatly simplifies rate calculation by providing direct, queryable fields for each interaction type, rather than relying on parsing Tasks or indirect `CampaignMember` status updates for all metrics.

## 3. Calculation Logic (Conceptual)

*   **Numerator - Identifying "Responses":**
    *   **Preferred:** Count unique `ContactId`/`LeadId` from `CampaignMember` where `Status` reflects a valid reply (e.g., 'Responded - Positive', 'Responded - General', 'Responded - Negative').
    *   **Alternative (Task-based):** Count unique `WhoId` from inbound reply `Task` records that have been successfully linked to an original outreach `Task` and categorized as a valid reply.
    *   **Alternative (Email_Interaction__oc):** Count unique `Contact__c` where `Response_Received_DateTime__c IS NOT NULL` and `Campaign__c` matches.

*   **Numerator - Identifying "Positive Responses," "Conversions," etc.:**
    *   Similar to above, but filter by more specific `CampaignMember.Status` values (e.g., 'Responded - Positive', 'Converted - Demo Scheduled') or `Email_Interaction__oc.Response_Sentiment__c = 'Positive'`.

*   **Denominator - "Number of Emails Delivered":**
    *   `COUNT_DISTINCT(CampaignMember.ContactId WHERE CampaignMember.CampaignId = 'X' AND CampaignMember.Status NOT IN ('Bounced - Hard', 'Bounced - Soft', 'Targeted', 'Scheduled'))`.
    *   Or, `Campaign.NumberOfSentEmails` (if accurate and represents attempts) minus the count of bounced `CampaignMember` records.
    *   From `Email_Interaction__oc`: `COUNT_DISTINCT(Contact__c WHERE Sent_DateTime__c IS NOT NULL AND Bounced_DateTime__c IS NULL AND Campaign__c = 'X')`.

*   **Example: Reply Rate (using CampaignMember)**
    ```
    Num_Unique_Contacts_Responded = COUNT_DISTINCT(CM.ContactId WHERE CM.CampaignId = 'XYZ' AND CM.Status IN ('Responded - General', 'Responded - Positive', 'Responded - Negative', 'Responded - Neutral'))
    Num_Emails_Delivered = COUNT_DISTINCT(CM.ContactId WHERE CM.CampaignId = 'XYZ' AND CM.Status NOT IN ('Bounced - Hard', 'Bounced - Soft', 'Targeted', 'Scheduled'))
    Reply_Rate = (Num_Unique_Contacts_Responded / Num_Emails_Delivered) * 100
    ```

## 4. Reporting and Logging

*   **a. Fields on the `Campaign` Object:**
    *   **Method:** Use custom fields. These can be populated by:
        1.  **Roll-Up Summary Fields:** If `CampaignMember` is master-detail to `Campaign` (not standard). Limited to counts and simple sums.
        2.  **Apex Triggers:** On `CampaignMember` (or `Task`, `Email_Interaction__oc`) to update aggregate fields on the parent `Campaign` in near real-time. Complex for distinct counts.
        3.  **Scheduled Batch Apex:** Nightly or hourly jobs to calculate and update these fields. Most scalable for complex calculations and distinct counts.
    *   **Example Fields:** `Reply_Rate_Calc__c` (Percent), `Positive_Response_Rate_Calc__c` (Percent), `Total_Delivered_Calc__c` (Number).
    *   **Logging Implication:** These fields store a snapshot of the rates, updated based on the frequency of the calculation logic (trigger or batch).

*   **b. Salesforce Reports and Dashboards:**
    *   **Primary Viewing Method:** Dynamically calculate and display metrics using Salesforce's reporting engine.
    *   **Report Types:**
        *   "Campaigns with Campaign Members" is standard.
        *   Custom Report Types if using `Email_Interaction__oc` or complex `Task` relationships.
    *   **Report Formulas:** Use summary formulas for rates (e.g., `CampaignMember.HasResponded:SUM / RowCount`). Group by `Campaign.Name`.
    *   **Dashboards:** Visualize report data with charts for trends and comparisons.
    *   **Logging Implication:** Reports are dynamic; they calculate rates when run. For historical "logging," report snapshots can be scheduled (e.g., "Reporting Snapshots" feature to store aggregated data over time in a custom object, or manual/API-based export).

*   **c. AI's Internal Data Store / External System:**
    *   The AI can periodically query Salesforce (using SOQL based on the logic above) to fetch raw interaction data or pre-calculated rates from Campaign fields/reports.
    *   This data can be stored externally for longitudinal analysis, cross-campaign performance comparisons, and training the AI's decision-making models regarding effective outreach.

**AI's Role in Data Integrity:**
*   **Reply Categorization:** AI plays a vital role in parsing email replies (`Task.Description`) to distinguish genuine human responses from auto-replies (Out of Office, etc.) and to categorize sentiment/intent.
*   **Status Updates:** Based on this analysis, the AI should drive updates to `CampaignMember.Status` or fields in `Email_Interaction__oc` to ensure data accuracy for reporting.

This comprehensive approach allows for flexible and scalable tracking of outreach campaign effectiveness. The choice between relying on `CampaignMember.Status` updates driven by AI analysis of `Task` records versus implementing a dedicated `Email_Interaction__oc` object depends on the desired granularity and complexity tolerance.
