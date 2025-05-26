# AI-Driven Lead/Account Prioritization for Restaurant Onboarding

This document outlines the logic for an AI system to categorize the priority (High, Medium, Low) of restaurant `Account` records for onboarding focus. Prioritization is based on key restaurant attributes to help sales and onboarding teams manage their efforts effectively.

## 1. Input Data from Salesforce

*   **Source Object:** `Account`
    *   This logic is primarily applied to `Account` records that are newly created, recently converted from `Lead`, or are in an early stage of the onboarding pipeline (e.g., linked to a `Restaurant_Onboarding__c` record in a stage like 'New Signup / Pending Initiation' or 'Information Gathering').
*   **Key Input Fields from `Account`:**
    *   `Account.Restaurant_Size__c` (Picklist): Indicates the capacity or scale of the restaurant (e.g., 'Small (1-20 seats)', 'Medium (21-60 seats)', 'Large (61+ seats)', 'Food Truck', 'Ghost Kitchen').
    *   `Account.Cuisine_Type__c` (Picklist or Text): Specifies the type of cuisine offered (e.g., 'Italian', 'Mexican', 'Indian', 'Chinese', 'Cafe', 'Fine Dining', 'Quick Service Restaurant (QSR)'). If this is a Text field, pre-processing or robust AI interpretation might be needed to categorize it.
    *   `Account.Current_Ordering_System__c` (Text): Describes the restaurant's existing online ordering setup, if any (e.g., 'None', 'Internal Website Only', 'Uses DoorDash', 'Legacy System XYZ', 'Modern POS with Online Ordering').

## 2. AI Processing Logic (Conceptual)

The AI can determine priority using one of the following conceptual approaches:

### Option A: Rule-Based / Decision Matrix Approach

This method uses a predefined set of rules or a decision matrix. It is transparent, directly controllable, and suitable for well-understood prioritization criteria.

*   **Sample Decision Matrix / Rules:**
    *(Note: These rules are illustrative and must be configured based on specific business strategy and market focus.)*

    | Priority | `Restaurant_Size__c`         | `Cuisine_Type__c` (Examples)      | `Current_Ordering_System__c` (Examples)                     | Rationale (Implied by Rule)                                                                          |
    | :------- | :--------------------------- | :-------------------------------- | :---------------------------------------------------------- | :--------------------------------------------------------------------------------------------------- |
    | **High** | 'Large (61+ seats)'          | Any                               | 'None', 'Legacy System X', 'Fax Orders', 'Phone Only'       | Large potential, high immediate need for a modern system.                                            |
    | **High** | 'Medium (21-60 seats)'       | Any                               | 'None', 'Legacy System X'                                   | Significant potential, clear need.                                                                   |
    | **High** | Any                          | 'Pizza', 'Burgers', 'Wings' (High Volume QSR) | 'None', 'Legacy System X'                                   | High volume categories with proven fit for online ordering.                                          |
    | **Medium** | 'Large (61+ seats)'          | Any                               | Uses a single major 3rd party (e.g., 'DoorDash Only')       | High potential but may require displacing an existing provider.                                        |
    | **Medium** | 'Medium (21-60 seats)'       | 'QSR', 'Cafe', 'Fast Casual'      | Any system, or 'None'                                       | Good fit for standard onboarding, moderate volume.                                                   |
    | **Medium** | 'Small (1-20 seats)'         | 'QSR', 'Cafe', 'Food Truck'       | 'None', 'Internal Website (no online order)'                | Easier to onboard, good entry point if product is suitable.                                          |
    | **Low**    | Any                          | Any                               | 'Custom API', 'Modern POS with Integrated Online Ordering'  | Complex integration or low immediate need if current system is advanced and satisfactory.            |
    | **Low**    | 'Small (1-20 seats)'         | 'Fine Dining', Highly Niche       | Any                                                         | May have very specific low-volume needs or prefer traditional methods.                               |
    | **Low**    | *Data Missing for >1 Key Field* | -                                 | -                                                           | Cannot accurately prioritize; requires data enrichment first. (Alternatively, could be 'Medium').    |

*   **Logic Flow:**
    1.  The AI retrieves the `Restaurant_Size__c`, `Cuisine_Type__c`, and `Current_Ordering_System__c` fields from an `Account` record.
    2.  The values are evaluated against the predefined rules, typically in a specific order of precedence.
    3.  The first matching rule determines the priority.
    4.  If crucial data is missing (e.g., `Restaurant_Size__c` is NULL), a default priority (e.g., 'Low' or 'Medium - Needs Review') is assigned as per the defined handling for missing data.

### Option B: LLM-Based Approach

This method uses a Large Language Model (LLM) to interpret the input data within the context of a detailed prompt that outlines prioritization guidelines.

*   **Prompt Structure:**
    The AI system constructs a prompt to the LLM, including the restaurant's data and clear instructions.
*   **Example Prompt Snippet:**
    ```
    "As an AI specializing in restaurant technology adoption, please assign an onboarding priority (High, Medium, or Low) to the following restaurant prospect. Also, provide a concise, one-sentence rationale for your decision.

    Restaurant Information:
    - Size: '[Account.Restaurant_Size__c]'
    - Cuisine Type: '[Account.Cuisine_Type__c]'
    - Current Online Ordering System: '[Account.Current_Ordering_System__c]'

    Prioritization Guidelines:
    - **High Priority:** Restaurants that are large, have no current online ordering system, use a very outdated system (e.g., 'Phone orders only', 'Fax machine'), or are high-volume cuisine types (like Pizza, QSR) without an efficient system. Also, consider high if they are part of a chain or group we are strategically targeting.
    - **Medium Priority:** Medium-sized restaurants, those using some third-party delivery platforms (but potentially interested in direct ordering), or smaller restaurants with popular, easy-to-onboard cuisine types.
    - **Low Priority:** Very small or niche restaurants with low order volume expectations, those with highly complex or custom-built existing systems that are difficult to integrate with, or those indicating no immediate interest in changing their current setup. If essential data like 'Restaurant Size' is missing, note this and assign 'Low' or 'Medium' as appropriate, suggesting data collection.

    Desired Output Format:
    Priority: [High/Medium/Low]
    Rationale: [Your one-sentence rationale]
    "
    ```
*   **Logic Flow:**
    1.  The AI retrieves the relevant fields from the `Account`.
    2.  It constructs the detailed prompt, inserting the Account's specific data.
    3.  The prompt is sent to the LLM via API.
    4.  The LLM's response (text) is parsed to extract the "Priority" value and the "Rationale".

## 3. Output

*   **Categorization:** A single string value: "High", "Medium", or "Low".
*   **Rationale (Optional, but recommended with LLM approach):** A brief text string explaining the AI's reasoning.
    *   Example (from LLM): "High: This is a large restaurant currently using only phone orders, indicating a significant opportunity for impact with a new online system."

## 4. Action in Salesforce

*   **Target Field for Priority Update:**
    *   `Account.AI_Assigned_Priority__c` (Picklist: 'High', 'Medium', 'Low'). The AI's determined priority will be written to this field.
*   **Target Field for Rationale Storage (if using LLM or if rule-based system generates it):**
    *   A new custom field, such as `Account.AI_Priority_Rationale__c` (Text Area or Long Text Area), should be created to store the textual rationale. This provides transparency and context for human reviewers.

## 5. Considerations

*   **Handling Missing Input Data:**
    *   **Rule-Based:** Explicit rules should define outcomes if key data points are NULL (e.g., assign 'Medium' and flag for data enrichment, or directly assign 'Low').
    *   **LLM-Based:** The prompt should guide the LLM on how to handle missing data (as shown in the example prompt). The AI system can also have pre-LLM checks to flag records with insufficient data for prioritization.
*   **Configurability and Learning/Tuning:**
    *   **Rule-Based:** The rule set should be easily accessible and modifiable by business administrators as strategies change (e.g., using Custom Metadata Types in Salesforce to store rules).
    *   **LLM-Based:**
        *   **Prompt Engineering:** The quality of the prompt is crucial. Iterative refinement will be necessary.
        *   **Feedback Mechanism:** A system allowing users to rate or correct AI-assigned priorities can be valuable for ongoing prompt improvement or for fine-tuning a dedicated model.
        *   **Cost and Latency:** API calls to LLMs have associated costs and can introduce latency. Consider batch processing for large numbers of accounts rather than real-time prioritization for every minor update.
*   **Data Normalization:**
    *   For text fields like `Account.Current_Ordering_System__c` or if `Account.Cuisine_Type__c` is text, the AI might need a pre-processing step to normalize or categorize these values to ensure consistent interpretation by rules or the LLM.
*   **Hybrid Approaches:**
    *   A combination might be effective: e.g., use rules for clear-cut high/low priority cases and an LLM for more ambiguous situations or to generate nuanced rationales.
*   **Initial Run and Backfill:**
    *   When first implemented, a process will be needed to run this AI prioritization logic on existing relevant Accounts.

This AI-driven prioritization logic aims to provide a consistent and data-informed approach to focusing onboarding efforts on the most promising restaurant accounts.Okay, the markdown document `ai_logic_lead_prioritization.md` has been created.

It covers:
1.  **Input Data from Salesforce:** Specifies `Account` as the source object and lists `Restaurant_Size__c`, `Cuisine_Type__c`, and `Current_Ordering_System__c` as key input fields.
2.  **AI Processing Logic (Conceptual):**
    *   **Option A: Rule-Based/Decision Matrix Approach:** Includes a sample decision matrix with example rules demonstrating how combinations of input fields lead to High, Medium, or Low priority, along with implied rationales.
    *   **Option B: LLM-Based Approach:** Defines the structure of a prompt for an LLM, including placeholders for the input data, guidelines for prioritization, and the desired output format (Priority and Rationale).
3.  **Output:** Details the expected output: the "High", "Medium", or "Low" categorization, and an optional brief rationale, especially if using an LLM.
4.  **Action in Salesforce:** Specifies `Account.AI_Assigned_Priority__c` as the target field for the priority and suggests a new custom field like `Account.AI_Priority_Rationale__c` for storing any generated rationale.
5.  **Considerations:** Discusses handling missing input data, the need for configurability or tuning (for both rule-based and LLM approaches), data normalization for text fields, potential for hybrid approaches, and the need for an initial run on existing records.

This document should provide a solid foundation for implementing the AI-driven prioritization logic.
