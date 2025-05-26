# Salesforce Objects and Fields for Restaurant Onboarding

This document outlines the Salesforce data model specifically for managing the "Restaurant Onboarding" lifecycle. It details the standard and custom objects, along with their key fields, that will support the AI assistant and the overall onboarding process.

## 1. Standard Objects in Restaurant Onboarding

*   **Lead:**
    *   **Purpose:** Represents potential restaurants (prospects) that have expressed initial interest or have been identified through marketing efforts but are not yet qualified.
    *   **Key Standard Fields:**
        *   `Company`: Name of the restaurant.
        *   `FirstName`, `LastName`: Contact person at the restaurant.
        *   `Email`, `Phone`: Contact details.
        *   `Street`, `City`, `State`, `PostalCode`, `Country`: Restaurant address.
        *   `LeadSource`: How the lead was generated (e.g., 'Web', 'Referral', 'Trade Show').
        *   `Status`: Lead lifecycle stage (e.g., 'Open - Not Contacted', 'Working - Contacted', 'Qualified', 'Unqualified').
        *   `Description`: General notes, initial interest details.
    *   **Custom Fields (Examples):**
        *   `Lead.Cuisine_Type_Inquiry__c` (Text): Cuisine type mentioned during initial inquiry.
        *   `Lead.Estimated_Restaurant_Size_Inquiry__c` (Text): Size mentioned or estimated.
    *   **Usage:** Leads are qualified by sales/AI. Upon qualification, a Lead is converted into an `Account`, `Contact`, and optionally an `Opportunity`.

*   **Account (Represents the Restaurant Entity):**
    *   **Purpose:** The primary record for each restaurant. This object holds core information about the restaurant business.
    *   **Key Standard Fields:**
        *   `Name`: Official name of the restaurant.
        *   `Phone`: Main phone number.
        *   `Website`: Restaurant's website.
        *   `BillingStreet`, `BillingCity`, `BillingState`, `BillingPostalCode`, `BillingCountry`: Billing address (may differ from physical location if part of a group).
        *   `ShippingStreet`, `ShippingCity`, `ShippingState`, `ShippingPostalCode`, `ShippingCountry`: Physical location of the restaurant.
        *   `Industry`: Should be set to "Hospitality" or "Food & Beverage".
        *   `Ownership`: e.g., 'Private', 'Franchise'.
    *   **Key Custom Fields (Restaurant Specific):**
        *   `Account.Cuisine_Type__c` (Picklist or Multi-select Picklist): e.g., 'Italian', 'Mexican', 'Indian', 'Chinese', 'Cafe', 'Fine Dining'.
        *   `Account.Restaurant_Size__c` (Picklist): e.g., 'Small (1-20 seats)', 'Medium (21-60 seats)', 'Large (61+ seats)', 'Food Truck', 'Ghost Kitchen'. (Alternatively, `Estimated_Covers_Per_Day__c` (Number)).
        *   `Account.Current_Ordering_System__c` (Text): Free-text field to note any existing online ordering systems they use.
        *   `Account.Point_of_Sale_System__c` (Picklist or Text): If common POS systems are known, a picklist is better. Otherwise, text. e.g., 'Square', 'Toast', 'Clover', 'Custom'.
        *   `Account.Primary_Contact__c` (Lookup to Contact): Designates the main point of contact at the restaurant.
        *   `Account.Service_Tier__c` (Picklist): e.g., 'Basic', 'Premium', 'Enterprise'.
        *   `Account.Activation_Date__c` (Date): Date when the restaurant officially goes live on the platform.
        *   `Account.AI_Assigned_Priority__c` (Number or Picklist): Priority assigned by AI for onboarding attention (e.g., 1-5 or High/Medium/Low).
        *   `Account.Missing_Onboarding_Data_Flags__c` (Text Area or Multi-select Picklist): To flag what critical information is still missing (e.g., 'Menu Missing', 'Banking Details Missing', 'Documents Pending'). AI can help populate this.

*   **Contact (Represents Individuals at the Restaurant):**
    *   **Purpose:** Stores information about people associated with the restaurant Account (owners, managers, chefs, finance contacts).
    *   **Key Standard Fields:**
        *   `AccountId` (Lookup to Account): Links the contact to the restaurant.
        *   `FirstName`, `LastName`, `Email`, `Phone`, `MobilePhone`, `Title`.
        *   `HasOptedOutOfEmail`: Standard opt-out field.
    *   **Key Custom Fields:**
        *   `Contact.Role_At_Restaurant__c` (Picklist): e.g., 'Owner', 'Manager', 'Chef', 'Finance Contact', 'Technical Contact'.
        *   `Contact.Receives_Onboarding_Comms__c` (Checkbox): Indicates if this contact should receive onboarding-related communications.

*   **Opportunity (Optional, for Tracking Sign-ups as Deals):**
    *   **Purpose:** Can be used to represent the "deal" of signing up a new restaurant, especially if there's a sales process or contract negotiation involved before onboarding formally begins. If onboarding starts immediately after a simple sign-up, an Opportunity might be overkill, and the `Restaurant_Onboarding__c` record can be created directly.
    *   **Key Standard Fields:**
        *   `AccountId` (Lookup to Account): The restaurant being signed up.
        *   `Name`: e.g., "[Restaurant Name] - New Signup".
        *   `StageName`: Sales lifecycle (e.g., 'Prospecting', 'Qualification', 'Needs Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won', 'Closed Lost').
        *   `CloseDate`: Expected date to close the deal (restaurant goes live or contract signed).
        *   `Amount`: Potential value of the signup (if applicable).
    *   **Usage:** When an Opportunity `StageName` reaches 'Closed Won', it can trigger the creation of a `Restaurant_Onboarding__c` record.

## 2. Custom Object: `Restaurant_Onboarding__c`

*   **Purpose:** This is the central custom object to track the detailed progress of a single restaurant through all stages of the onboarding lifecycle, from initial agreement/signup to going live. It acts as a case or project management tool for each onboarding instance.
*   **Relationships:**
    *   `Account__c` (Master-Detail or Lookup to `Account`): Links the onboarding process to the specific restaurant. Master-Detail is often preferred if the onboarding record should not exist without an Account and for easier rollup summaries.
*   **Key Custom Fields:**
    *   `Onboarding_ID__c` (Auto-Number/Text): A unique identifier for the onboarding process.
    *   `Onboarding_Stage__c` (Picklist): Defines the current stage of the onboarding process. Values are critical for tracking and AI prompts:
        *   'New Signup / Pending Initiation'
        *   'Initial Contact Made'
        *   'Information Gathering' (e.g., for KYC, operational details)
        *   'Documents Requested'
        *   'Documents Submitted'
        *   'Documents Verified'
        *   'Menu Collection Pending'
        *   'Menu Submitted / Processing'
        *   'Menu Digitization / Setup'
        *   'Menu Approved by Restaurant'
        *   'Platform Profile Setup Pending'
        *   'Platform Profile Complete'
        *   'Banking Setup Pending'
        *   'Banking Details Submitted'
        *   'Banking Details Verified'
        *   'Integration Setup (POS/Other)' (If applicable)
        *   'Training Scheduled'
        *   'Training Completed'
        *   'Ready for Activation Review'
        *   'Activation Approved / Go-Live Scheduled'
        *   'Live'
        *   'Post-Live Check-in'
        *   'Onboarding Complete'
        *   'Onboarding Halted / Cancelled'
    *   `Onboarding_Status__c` (Picklist): Provides further detail on the current state, e.g., 'On Track', 'At Risk', 'Delayed', 'Needs Attention', 'Awaiting Restaurant Action'. AI can help set this.
    *   `Onboarding_Owner__c` (Lookup to `User`): The internal specialist responsible for this restaurant's onboarding.
    *   `Stage_Entry_Date__c` (Date/Time): Timestamp when the current `Onboarding_Stage__c` was entered. Useful for SLA tracking and identifying bottlenecks.
    *   `Kick_Off_Date__c` (Date): Date onboarding process officially started.
    *   `Projected_Go_Live_Date__c` (Date): Estimated date for the restaurant to go live.
    *   `Actual_Go_Live_Date__c` (Date): The date the restaurant actually went live.
    *   `Documents_Portal_Link__c` (URL): Link to a shared folder/portal (e.g., SharePoint, Google Drive) for document exchange.
    *   `Missing_Information_Detail__c` (Long Text Area): Detailed notes on what specific information or actions are pending from the restaurant or internally. AI can populate this.
    *   `Next_Step_AI__c` (Text Area): AI-suggested next best action for the onboarding owner.
    *   `AI_Onboarding_Priority__c` (Number or Picklist): Overrides Account-level priority if needed for this specific onboarding instance.
    *   `Menu_Submission_Date__c` (Date)
    *   `Menu_Approval_Date__c` (Date)
    *   `Banking_Submission_Date__c` (Date)
    *   `Banking_Approval_Date__c` (Date)
    *   `Documents_Submission_Date__c` (Date)
    *   `Documents_Approval_Date__c` (Date)
    *   `Time_In_Current_Stage_Days__c` (Formula, Number): Calculates days spent in the current stage using `Stage_Entry_Date__c`.
    *   `Total_Onboarding_Duration_Days__c` (Formula, Number): Calculates total days from `Kick_Off_Date__c` to `Actual_Go_Live_Date__c` or today if not live.
    *   `Onboarding_Notes__c` (Long Text Area): General notes and history for the onboarding process.

## 3. Activities (Tasks and Events)

*   **Standard `Task` and `Event` objects** will be used extensively, related to `Account`, `Contact`, and `Restaurant_Onboarding__c` (via `WhatId` or `RelatedTo` fields).
*   **Purpose:** To log calls, emails, meetings, internal follow-ups, document collection efforts, menu review sessions, etc.
*   **Key Custom Fields on `Activity` (Task/Event) (Examples):**
    *   `Activity.Onboarding_Related_To_Stage__c` (Picklist, values mirror `Restaurant_Onboarding__c.Onboarding_Stage__c`): To categorize activities by the onboarding stage they occurred in or relate to.
    *   `Activity.AI_Generated_Summary__c` (Long Text Area): For AI to summarize call transcripts or email threads.

## 4. Files / Attachments / Content

*   **Standard `File` / `Attachment` / `ContentDocument` objects** will be used to store:
    *   Signed contracts
    *   Restaurant menus (various versions)
    *   Business licenses, permits
    *   Photos for the platform profile
    *   Banking information documents (securely handled as per Salesforce Shield / security guidelines if applicable)
*   These will be related to `Account`, `Contact`, or `Restaurant_Onboarding__c` records.

## 5. Considerations for AI Prompts & Automation

*   **Data Completeness:** Fields like `Account.Missing_Onboarding_Data_Flags__c` and `Restaurant_Onboarding__c.Missing_Information_Detail__c` directly feed AI prompts for requesting missing info.
*   **Prioritization:** `Account.AI_Assigned_Priority__c` and `Restaurant_Onboarding__c.AI_Onboarding_Priority__c` guide AI focus.
*   **Next Best Action:** `Restaurant_Onboarding__c.Next_Step_AI__c` is where the AI logs its suggestions.
*   **Stage Progression:** The AI will monitor key date fields and status changes on `Restaurant_Onboarding__c` to trigger stage updates or alerts.
*   **Communication Triggers:** Changes in `Onboarding_Stage__c` or `Onboarding_Status__c` can trigger AI-assisted communication with the restaurant contact or internal owner.
*   **Bottleneck Identification:** AI can analyze `Time_In_Current_Stage_Days__c` across multiple onboarding records to identify common bottlenecks.

This data model provides a comprehensive foundation for building a robust restaurant onboarding application within Salesforce, with clear roles for standard and custom objects and fields designed to support AI-driven assistance and automation.Okay, I have updated the `salesforce_object_outline.md` file with the new content focusing on the "Restaurant Onboarding" lifecycle.

The file now includes:
1.  **Standard Objects:** Details on how `Lead`, `Account`, `Contact`, and `Opportunity` are utilized in restaurant onboarding.
2.  **Custom Object `Restaurant_Onboarding__c`:** Elaboration on its structure, purpose, relationship to Account, and key custom fields including:
    *   `Onboarding_Stage__c` (with a comprehensive picklist of stages)
    *   `Onboarding_Owner__c`
    *   `Account__c`
    *   Various date fields for tracking stage progression (e.g., `Stage_Entry_Date__c`, `Menu_Submitted_Date__c`).
3.  **Key Custom Fields on `Account`:**
    *   `Cuisine_Type__c`
    *   `Restaurant_Size__c`
    *   `Current_Ordering_System__c`
    *   `Point_of_Sale_System__c`
    *   `AI_Assigned_Priority__c`
    *   `Missing_Onboarding_Data_Flags__c`
4.  **Other relevant objects and fields:** Use of Activities (Tasks/Events) and Files/Attachments, with considerations for AI prompts (data completeness, prioritization, next best action, stage progression, communication triggers, bottleneck identification).

The document should now serve as a comprehensive data model guide for the restaurant onboarding application.
