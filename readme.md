# **Looker \+ Gemini: The AI-Powered Semantic Layer Chatbot**

This project is a multi-agent Streamlit application demonstrating how to build an AI chatbot on top of Looker's semantic layer to answer complex data and analytical questions.

The system uses a **Steering Agent** to route user queries to specialized agents, including the **Looker Data Agent** for census questions and the **Python Agent** for data analysis on cached results.

## **🚀 Deployment Guide**

Follow these steps to deploy and run the application locally or on a cloud platform like Google Cloud Run.

### **Step 1: Set Up Credentials**

The application uses Streamlit Secrets to manage API keys. Create a folder named .streamlit in the project root and add a file named secrets.toml with the following structure:

\# .streamlit/secrets.toml

\# Google Gemini API Key  
GOOGLE\_API\_KEY \= "YOUR\_GEMINI\_API\_KEY"

\# Looker SDK Credentials (used by looker\_tool.py)  
\[looker\]  
base\_url \= "\[https://yourinstance.cloud.looker.com:19999\](https://yourinstance.cloud.looker.com:19999)" \# Note: Ensure you include the :19999 for API calls  
client\_id \= "YOUR\_LOOKER\_CLIENT\_ID"  
client\_secret \= "YOUR\_LOOKER\_CLIENT\_SECRET"

\# Proxycurl API Key (used by social\_tool.py)  
\# NOTE: This tool is used by the Social Agent and can be removed if not needed.  
PROXYCURL\_API\_KEY \= "YOUR\_PROXYCURL\_API\_KEY"

### **Step 2: Install Dependencies**

The project uses several Python libraries for the core functionality, as specified in requirements.txt.

1. **Create a virtual environment** (recommended):  
   python \-m venv venv  
   source venv/bin/activate  \# On Linux/macOS  
   .\\venv\\Scripts\\activate   \# On Windows

2. **Install packages** (using the provided requirements.txt):  
   pip install \-r requirements.txt

### **Step 3: Generate Looker Metadata (Mandatory)**

The tools/looker\_tool.py and tools/knowledge\_tool.py agents rely on a local file named acs\_census\_metadata.json (or your chosen name) to understand the available dimensions and measures.

The script 01\_fetch\_metadata.py is used to generate this file using the Looker SDK.

1. **Ensure Looker SDK is configured** (it uses the variables in secrets.toml or a local looker.ini file).  
2. **Run the script** (This assumes you are using the same Model/Explore names as the original project: data\_block\_acs\_bigquery::acs\_census\_data):  
   python 01\_fetch\_metadata.py

   This step generates the acs\_census\_metadata.json file, which is then loaded by the Looker tool.

### **Step 4: Run the Application**

Start the Streamlit application from the root directory:

streamlit run app.py

The application will launch in your default web browser, allowing you to interact with the multi-agent chatbot.

## **🛠️ Customization Guide: Using a Different Looker Explore**

To adapt this chatbot to a different Looker Explore (e.g., your internal financial model instead of the US Census Data), you need to modify three core files.

### **1\. Update Looker Model and Explore Names**

Edit tools/looker\_tool.py to target your new data source.

| File: tools/looker\_tool.py | Change From | Change To (Example) |
| :---- | :---- | :---- |
| MODEL\_NAME | "data\_block\_acs\_bigquery" | "your\_custom\_model" |
| EXPLORE\_NAME | "acs\_census\_data" | "your\_custom\_explore" |

### **2\. Generate New Metadata JSON**

You must generate a new metadata file containing the fields from your custom Explore.

1. **Update 01\_fetch\_metadata.py**:  
   * Change MODEL\_NAME and EXPLORE\_NAME in this file to match the values from the step above.  
   * Change output\_filename to something descriptive, e.g., "your\_explore\_metadata.json".  
2. **Run the updated script**:  
   python 01\_fetch\_metadata.py

3. This creates your new metadata file (e.g., your\_explore\_metadata.json).

### **3\. Update Metadata Loading References**

Update the Looker tool and the Knowledge tool to load your new metadata file.

* **In tools/looker\_tool.py**: Modify the \_get\_explore\_metadata function to reference your new JSON file:  
  \# tools/looker\_tool.py  
  def \_get\_explore\_metadata():  
      """Loads the explore metadata JSON file."""  
      try:  
          \# CHANGE FILENAME HERE  
          with open("your\_explore\_metadata.json") as f:  
              return json.dumps(json.load(f))  
          ...

* **In tools/knowledge\_tool.py**: Modify the get\_census\_data\_definition function to load the correct file:  
  \# tools/knowledge\_tool.py  
  @tool(args\_schema=TermInput)  
  def get\_census\_data\_definition(term: str) \-\> str:  
      \# CHANGE FILENAME HERE  
      with open('your\_explore\_metadata.json', 'r') as f:  
          metadata \= json.load(f)  
      ...



  
  sequenceDiagram
      participant User as Google Chat User
      participant Bot as EAP Chat Bot<br/>(Cloud Run Function Gen 2)<br/>Project: cte-dse-g-chat-bot-np
      participant LLM as Gemini 2.5-Flash<br/>(LangGraph ReAct Agent)
      participant Meta as GCP Metadata Server<br/>(metadata.google.internal)
      participant Google as Google Auth<br/>(accounts.google.com)
      participant IAP as Identity-Aware Proxy<br/>(Org brand: project 369001918367)
      participant Agent as DataHub 2.0 Pipeline Agent<br/>(Cloud Run - Skipper A2A)<br/>Project: cio-eap-n8n-automation-np

      Note over User, Agent: 1. USER SENDS MESSAGE

      User->>Bot: "What pipelines exist in DataHub?"
      Bot->>LLM: invoke(messages)
      LLM->>LLM: ReAct reasoning:<br/>"This is about DataHub 2.0,<br/>I should use pipeline_agent tool"
      LLM-->>Bot: Tool call: pipeline_agent("What pipelines exist?")

      Note over Bot, Agent: 2. JWT TOKEN GENERATION

      Bot->>Bot: _get_iap_headers() called
      Bot->>Bot: Check PIPELINE_AGENT_IAP_CLIENT_ID env var

      alt Primary: id_token.fetch_id_token (Gen 2)
          Bot->>Meta: GET
  /computeMetadata/v1/instance/<br/>service-accounts/default/identity<br/>?audience=369001918367-...apps.googleusercontent.com<br/>Header: Metadata-Flavor:
  Google
          Meta->>Google: "SA g-chat-service-account@...<br/>needs ID token for audience 369001918367-..."
          Google->>Google: Verify SA identity<br/>Create JWT:<br/>  iss: accounts.google.com<br/>  sub: 105633110857141712042<br/>  aud:
  369001918367-...<br/>  exp: +1 hour<br/>Sign with Google private key
          Google-->>Meta: Signed JWT (eyJhbG...)
          Meta-->>Bot: JWT token
      else Fallback: Metadata server directly
          Bot->>Meta: GET /computeMetadata/v1/instance/<br/>service-accounts/default/identity<br/>?audience=369001918367-...<br/>Header: Metadata-Flavor:
  Google
          Meta-->>Bot: JWT token
      end

      Bot->>Bot: Set header:<br/>Authorization: Bearer eyJhbG...

      Note over Bot, Agent: 3. A2A TASK CREATION (through IAP)

      Bot->>IAP: POST /a2a/tasks<br/>Authorization: Bearer eyJhbG...<br/>Body: {"message": "What pipelines exist?"}

      IAP->>IAP: Validate JWT token:<br/>1. Is signature valid? (Google-signed) ✓<br/>2. Is token expired? ✓<br/>3. Does "aud" match my client ID?<br/>
  Token aud: 369001918367-...<br/>   My ID: ???<br/>   ❌ MISMATCH

      IAP-->>Bot: HTTP 401<br/>Invalid IAP credentials:<br/>Invalid bearer token.<br/>Invalid JWT audience.

      Note over IAP, Agent: ❌ REQUEST NEVER REACHES PIPELINE AGENT

      Note over Bot, Agent: 4. IF IAP SUCCEEDS (expected flow)

      rect rgb(200, 255, 200)
          IAP->>Agent: Forward request<br/>(IAP adds X-Goog-IAP-JWT-Assertion header)
          Agent->>Agent: Create task<br/>task_id: abc-123<br/>status: pending
          Agent-->>IAP: {"task_id": "abc-123", "status": "pending"}
          IAP-->>Bot: HTTP 200

          Note over Bot, Agent: 5. POLLING LOOP

          loop Poll every 2 seconds (max 60 attempts)
              Bot->>IAP: GET /a2a/tasks/abc-123<br/>Authorization: Bearer eyJhbG...
              IAP->>Agent: Forward request
              Agent-->>IAP: {"status": "running", ...}
              IAP-->>Bot: HTTP 200
          end

          Bot->>IAP: GET /a2a/tasks/abc-123
          IAP->>Agent: Forward request
          Agent-->>IAP: {"status": "completed",<br/>"messages": [{"content": "Found 5 pipelines..."}]}
          IAP-->>Bot: HTTP 200
      end

      Note over User, Agent: 6. RESPONSE TO USER

      Bot->>LLM: Tool result: "Found 5 pipelines..."
      LLM->>LLM: Format response
      LLM-->>Bot: Final answer
      Bot->>User: "Here are the pipelines in DataHub 2.0:..."

  ---
  Where it breaks

  flowchart LR
      A[Our Bot] -->|"JWT token<br/>aud: 369001918367-..."| B{IAP Guard}
      B -->|"aud matches my ID?"| C{Check}
      C -->|"✓ Match"| D[Pipeline Agent]
      C -->|"✗ Mismatch"| E["❌ HTTP 401<br/>Invalid JWT audience"]

      style E fill:#ff6666,color:#fff
      style D fill:#66cc66,color:#fff
      style C fill:#ffcc00

### **4\. Adjust Agent Strategy Prompts (Highly Recommended)**

For optimal AI performance, update the hardcoded prompts in app.py to guide the agent toward using relevant fields in your new domain.

* **app.py**: Find the LOOKER\_AGENT\_PROMPT\_TEMPLATE and specifically update the **ANALYST STRATEGY** section. Guide the agent to think like an analyst in your new domain (e.g., recommend fields like orders.count, products.inventory\_level, or financials.revenue instead of census demographics).
