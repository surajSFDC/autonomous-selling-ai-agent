# Salesforce Objects and Fields for AI Assistant

This document outlines the Salesforce objects and fields, both standard and custom, relevant to the AI assistant's functionalities across various categories.

## 1. Customer Segmentation

*   **Standard Object: Account**
    *   `Name` (Standard Field, Text): Organization name.
    *   `Website` (Standard Field, URL): Organization's website.
    *   `Industry` (Standard Field, Picklist): Industry of the organization.
    *   `NumberOfEmployees` (Standard Field, Number): Size of the organization.
    *   `Last_Activity_Date__c` (Custom Field, DateTime): Stores the timestamp of the last significant interaction or "trip" taken by the organization. Used to identify organizations that haven’t taken a trip in the last X days.
    *   `Email_Domain__c` (Custom Field, Text): Extracted domain from primary contact's email (e.g., "company.com"). Used to help filter out freemail domains at an account level or analyze domain types.
    *   `Verified_Email_Organization__c` (Custom Field, Boolean): Indicates if the primary contact(s) for the organization have verified email addresses.
    *   `AccountSource` (Standard Field, Picklist): How the account was acquired.
    *   `Ownership` (Standard Field, Picklist): e.g., Public, Private.

*   **Standard Object: Contact**
    *   `FirstName` (Standard Field, Text)
    *   `LastName` (Standard Field, Text)
    *   `Email` (Standard Field, Email): Contact's email address.
    *   `Phone` (Standard Field, Phone)
    *   `Is_Freemail__c` (Custom Field, Boolean, Formula): Formula field evaluating the `Email` field to determine if it's from a known freemail provider (e.g., gmail.com, outlook.com).
    *   `AccountId` (Standard Field, Lookup to Account): Link to the organization.

## 2. Outreach

*   **Standard Object: Campaign**
    *   `CampaignName` (Standard Field, Text): Name of the outreach initiative (e.g., "Q3 New User Acquisition AI Drip").
    *   `Type` (Standard Field, Picklist): e.g., Email, Webinar, Conference.
    *   `Status` (Standard Field, Picklist): e.g., Planned, In Progress, Completed.
    *   `StartDate` (Standard Field, Date)
    *   `EndDate` (Standard Field, Date)

*   **Standard Object: CampaignMember**
    *   `CampaignId` (Standard Field, Lookup to Campaign)
    *   `LeadId` (Standard Field, Lookup to Lead)
    *   `ContactId` (Standard Field, Lookup to Contact)
    *   `Status` (Standard Field, Picklist): Tracks individual's progress in the campaign (e.g., Sent, Opened, Clicked, Responded, Unsubscribed). This is crucial for tracking email sends and responses.

*   **Standard Object: Activity (Task or Event)**
    *   `Subject` (Standard Field, Text): e.g., "Email: Initial Outreach", "Call: Follow-up".
    *   `Type` (Standard Field, Picklist): 'Email', 'Call', 'Meeting'.
    *   `Status` (Standard Field, Picklist): For Tasks: Not Started, In Progress, Completed, Waiting on someone else, Deferred. For Events: Planned, Held, Not Held.
    *   `Priority` (Standard Field, Picklist)
    *   `ActivityDate` (Standard Field, Date for Tasks) / `StartDateTime`, `EndDateTime` (Standard Fields for Events): Tracks when the email was sent or the call/meeting occurred.
    *   `Description` (Standard Field, Long Text Area): Can store email body, call notes, or response snippets.
    *   `WhoId` (Standard Field, Lookup to Contact or Lead): The person associated with the activity.
    *   `WhatId` (Standard Field, Lookup to Account, Opportunity, etc.): The record associated with the activity.

*   **Custom Object: Email_Interaction__oc** (Consider if Activity object is insufficient for detailed email tracking)
    *   `Contact__c` (Lookup to Contact)
    *   `Account__c` (Lookup to Account, Master-Detail or Lookup)
    *   `Campaign__c` (Lookup to Campaign)
    *   `Email_Subject__c` (Text)
    *   `Sent_DateTime__c` (DateTime)
    *   `Opened_DateTime__c` (DateTime, nullable)
    *   `Clicked_DateTime__c` (DateTime, nullable)
    *   `Response_Received_DateTime__c` (DateTime, nullable)
    *   `Response_Content__c` (Long Text Area, nullable)
    *   `Response_Sentiment__c` (Picklist: Positive, Neutral, Negative, N/A)

## 3. Engagement

*   **Standard Object: Activity (Task or Event)**
    *   As above, used for logging conversations (calls, meetings).
    *   `Description` (Standard Field, Long Text Area): Store detailed notes or transcripts of conversations.
    *   `AI_Conversation_Summary__c` (Custom Field, Long Text Area): AI-generated summary of the conversation logged in `Description`.

*   **Standard Object: Contact**
    *   `Decision_Maker_Status__c` (Custom Field, Picklist): e.g., Decision Maker, Influencer, User, Gatekeeper, Not Identified. To help identify decision-makers.
    *   `Engagement_Level__c` (Custom Field, Picklist or Number): e.g., High, Medium, Low, based on interactions.

*   **Custom Object: Conversation_Log__oc** (If Activity object is not granular enough for conversation details)
    *   `Contact__c` (Lookup to Contact)
    *   `Account__c` (Lookup to Account, Master-Detail or Lookup)
    *   `Interaction_DateTime__c` (DateTime)
    *   `Channel__c` (Picklist: Email, Chat, Call, Meeting)
    *   `Full_Transcript__c` (Long Text Area, if available and permissible)
    *   `AI_Generated_Summary__c` (Long Text Area)
    *   `Key_Topics_Discussed__c` (Text Area, tags or keywords)
    *   `Identified_Needs_Pain_Points__c` (Text Area)
    *   `Expressed_Interest_Level__c` (Picklist: High, Medium, Low, None)
    *   `Next_Steps_Agreed__c` (Text Area)

## 4. Onboarding

*   **Standard Object: Account**
    *   `First_Trip_Date__c` (Custom Field, Date): Date of the organization's first completed "trip" or key activation event.
    *   `Onboarding_Status__c` (Custom Field, Picklist): e.g., Not Started, Initial Contact, Demo Scheduled, Technical Setup, First Trip Pending, First Trip Completed, Fully Onboarded.

*   **Standard Object: Opportunity** (If "first trip" is part of a sales or trial conversion process)
    *   `StageName` (Standard Field, Picklist): Could include stages like "Trial Started", "Awaiting First Trip".
    *   `First_Trip_Completed_Date__c` (Custom Field, Date): On the Opportunity if it's a key milestone for closing the deal.

*   **Custom Object: Trip__oc** (If "trips" are a central concept needing detailed attributes beyond a date)
    *   `Account__c` (Lookup to Account, Master-Detail)
    *   `Trip_DateTime__c` (DateTime): Start time of the trip.
    *   `Trip_Duration_Minutes__c` (Number)
    *   `Trip_Type__c` (Picklist: e.g., Evaluation, Standard, Premium Feature X)
    *   `Is_First_Account_Trip__c` (Boolean, Workflow/Process Builder updated): Marks if this was the account's first trip.
    *   `Associated_User_Contact__c` (Lookup to Contact, optional)

## 5. Retention

*   **Standard Object: Case**
    *   `AccountId` (Standard Field, Lookup to Account)
    *   `ContactId` (Standard Field, Lookup to Contact)
    *   `Subject` (Standard Field, Text): Brief summary of the feedback or issue.
    *   `Description` (Standard Field, Long Text Area): Detailed feedback content.
    *   `Type` (Standard Field, Picklist): e.g., Feedback, Problem, Question, Feature Request.
    *   `Reason` (Standard Field, Picklist): Sub-category for the type.
    *   `Status` (Standard Field, Picklist): e.g., New, In Progress, Awaiting Customer Response, Resolved, Closed.
    *   `Origin` (Standard Field, Picklist): e.g., AI Assistant, Email, Phone, Web Form.
    *   `Priority` (Standard Field, Picklist)
    *   `Feedback_Sentiment__c` (Custom Field, Picklist: Positive, Neutral, Negative, Mixed): Can be manually set or AI-derived from `Description`.

*   **Custom Object: Feedback_Entry__oc** (If `Case` object is too process-heavy for simple feedback logging)
    *   `Account__c` (Lookup to Account)
    *   `Contact__c` (Lookup to Contact, optional)
    *   `Submission_DateTime__c` (DateTime)
    *   `Source_Channel__c` (Picklist: AI Assistant, In-App Survey, Email Feedback)
    *   `Feedback_Category__c` (Picklist: Compliment, Complaint, Suggestion, Usability Issue)
    *   `Feedback_Text__c` (Long Text Area)
    *   `Sentiment__c` (Picklist: Positive, Neutral, Negative) - Potentially AI-populated.
    *   `Related_Feature_Area__c` (Text or Picklist, optional)

## 6. Meta Prompts (Data for Summaries and Prioritization)

This category leverages data from the objects and fields detailed above. The AI would perform queries and aggregations on:

*   **Account Data:** `Last_Activity_Date__c`, `Verified_Email_Organization__c`, `Industry`, `NumberOfEmployees`, `Onboarding_Status__c`, `First_Trip_Date__c`, `AccountSource`.
*   **Contact Data:** `Decision_Maker_Status__c`, `Is_Freemail__c`, `Engagement_Level__c`.
*   **Activity/Conversation Data:** `Type`, `Status`, `ActivityDate` (`StartDateTime`), `AI_Conversation_Summary__c`, `Key_Topics_Discussed__c`, `Identified_Needs_Pain_Points__c`.
*   **Campaign & Outreach Data:** `CampaignMember.Status`, `Response_Sentiment__c` (from `Email_Interaction__oc`).
*   **Onboarding Data:** `Account.Onboarding_Status__c`, `Account.First_Trip_Date__c` or `Trip__oc.Is_First_Account_Trip__c`.
*   **Retention/Feedback Data:** `Case.Type`, `Case.Reason`, `Case.Status`, `Case.Feedback_Sentiment__c` or `Feedback_Entry__oc.Sentiment__c`.
*   **Opportunity Data (if applicable):** `StageName`, `Amount`, `CloseDate`, `NextStep`.

The AI assistant would use this data to:
*   Generate summaries of customer health or engagement.
*   Prioritize outreach based on activity signals (or lack thereof).
*   Identify accounts needing attention for retention.
*   Segment customers for targeted campaigns.
*   Provide context for conversations.

**Note on Custom Suffixes:**
*   `__c`: Standard Salesforce suffix for custom fields on standard or custom objects, and for the API name of custom objects.
*   `__oc`: Used here conceptually to distinguish proposed custom objects from standard objects during this planning phase. In an actual Salesforce implementation, custom objects also end in `__c` for their API names (e.g., `Email_Interaction__c`).

This outline provides a foundational structure. Specific field types, picklist values, and relationships might be refined during actual implementation based on more detailed requirements.
