# Salesforce Integration Strategy for Restaurant Onboarding

## 1. Core Principle

The integration of the AI Assistant with Salesforce for restaurant onboarding will prioritize leveraging standard Salesforce objects and functionalities, augmented by a central custom object, `Restaurant_Onboarding__c`, to track the onboarding lifecycle. This approach promotes stability, utilizes existing Salesforce features, and simplifies maintenance. Custom fields will be added as defined in `salesforce_object_outline.md` to capture specific data points necessary for the AI and the onboarding process.

## 2. Key Objects for Restaurant Onboarding

The AI Assistant will primarily interact with:

*   **Standard Objects:** `Lead`, `Account`, `Contact`, `Opportunity` (if used for deal tracking), and `Activity (Task & Event)`. These will be used for managing prospect data, core restaurant information, individual contacts, sales processes, and interaction logging.
*   **Custom Object: `Restaurant_Onboarding__c`:** This is the central object for managing the detailed progress of each restaurant's onboarding. The AI will heavily interact with this object to track stages, update statuses, log information, and suggest next actions.
*   **Custom Fields:** Key custom fields on `Account` (e.g., `Cuisine_Type__c`, `AI_Assigned_Priority__c`) and `Restaurant_Onboarding__c` (e.g., `Onboarding_Stage__c`, `Onboarding_Status__c`, `Missing_Information_Detail__c`, `Next_Step_AI__c`) will be critical for AI functionality.

## 3. Authentication Method

The AI Assistant will authenticate with Salesforce using **OAuth 2.0**. This ensures secure, token-based access without needing to store Salesforce credentials directly. The JWT Bearer Flow is recommended for server-to-server integration where the AI acts on behalf of a designated integration user.

## 4. Data Synchronization

*   **Reading Data from Salesforce:**
    *   The AI Assistant will use Salesforce REST/SOAP APIs, employing SOQL for targeted data retrieval (e.g., specific `Restaurant_Onboarding__c` records, related Account details, missing information flags).
    *   **For `Restaurant_Onboarding__c` and related critical updates (e.g., `Onboarding_Stage__c` changes, new restaurant communications):** The use of **Change Data Capture (CDC)** or **Platform Events** is highly recommended. This allows the AI Assistant to subscribe to data changes in Salesforce in near real-time. This is crucial for enabling the AI to react promptly to stage advancements, status changes, or newly available information on `Restaurant_Onboarding__c` records (and related Accounts/Contacts) without resorting to inefficient constant polling.

*   **Writing Data to Salesforce:**
    *   The AI Assistant will update Salesforce records (e.g., creating tasks for follow-ups, updating `Restaurant_Onboarding__c.Onboarding_Status__c`, populating `Restaurant_Onboarding__c.Missing_Information_Detail__c` or `Restaurant_Onboarding__c.Next_Step_AI__c`) in near real-time via Salesforce REST/SOAP APIs.
    *   Updates will typically be performed immediately after an AI processing event (e.g., email drafted, information analyzed, next step determined) to ensure Salesforce reflects the most current state and guidance.

*   **API Limits:**
    *   All API interactions will be designed with Salesforce API limits in mind. This includes optimizing SOQL queries, using composite requests where appropriate, and employing efficient data retrieval strategies (e.g., leveraging CDC/Platform Events to reduce polling).
    *   Careful monitoring of API usage will be essential.

## 5. AI Interaction Patterns

*   **Transactional Updates (Near Real-Time):**
    *   Many AI interactions will involve direct, immediate updates to Salesforce. Examples include:
        *   Logging a drafted email as a Task.
        *   Updating the `Restaurant_Onboarding__c.Next_Step_AI__c` field after analysis.
        *   Changing `Restaurant_Onboarding__c.Onboarding_Status__c` based on new information.
        *   Flagging missing data on `Account.Missing_Onboarding_Data_Flags__c`.
    *   These actions are typically event-driven (e.g., user input, CDC event received, scheduled check) and aim to keep Salesforce data current.

*   **Analytical Processing (Batch/Periodic):**
    *   Some AI functions will involve broader data analysis that may not require or be suitable for immediate, record-by-record updates. Examples include:
        *   **Bottleneck Analysis:** Analyzing `Time_In_Current_Stage_Days__c` across multiple `Restaurant_Onboarding__c` records to identify common delays.
        *   **Overall Onboarding Performance Reporting:** Aggregating data to assess overall efficiency, success rates, and areas for systemic improvement.
        *   **Trend Analysis for Prioritization:** Reviewing patterns across Accounts or Onboarding records to adjust `AI_Assigned_Priority__c`.
    *   These functions might involve:
        *   Scheduled batch data reads from Salesforce.
        *   Processing data within the AI's environment.
        *   Results might be stored back in Salesforce (e.g., on a summary custom object or dashboard) or used to refine AI models and strategies.

## 6. Data Consistency

Maintaining data consistency between the AI Assistant's operational understanding and Salesforce records is paramount.
*   Salesforce will be treated as the source of truth for all restaurant and onboarding data.
*   Updates made by the AI to Salesforce will be designed to be atomic and idempotent where possible.
*   Robust error handling and reconciliation mechanisms will be implemented to manage any discrepancies or failed updates, ensuring data integrity.

This strategy aims to create a robust, scalable, and secure integration between the AI Assistant and Salesforce, enabling seamless data flow and empowering the AI with the necessary context to effectively support the restaurant onboarding process.
