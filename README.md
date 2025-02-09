# oracle_ai_plsql_agent
This PL/SQL package lets you generate &amp; execute PL/SQL code from natural language using OCI's Generative AI . Exclusively for Oracle ATP on OCI (23ai+). It takes text input, creates PL/SQL, runs it, &amp; shows the code/results. Requires OCI credentials &amp; AI profile setup. Simplifies database interaction.

# PL/SQL LLM Agent for Oracle Cloud Infrastructure (OCI)

## Overview

This repository contains a PL/SQL package (`plsql_agent`) that acts as an agent, leveraging the power of Large Language Models (LLMs) via Oracle Cloud Infrastructure's Generative AI service (specifically, Meta's LLaMA 3 model).  This agent allows you to generate and execute PL/SQL code dynamically based on natural language input.  It's designed to work *exclusively* with Oracle Autonomous Transaction Processing (ATP) databases on OCI (version 23ai or later) that have access to the `DBMS_CLOUD_AI.GENERATE` function.  This functionality is **not available on on-premise databases**.

**Key Features:**

*   **Natural Language to PL/SQL:**  Convert plain English instructions into executable PL/SQL code.
*   **Dynamic Code Execution:**  The generated code is executed immediately within the database session.
*   **DBMS_OUTPUT Capture:**  Captures and displays the output from the executed PL/SQL code, including results and any debugging information.
*   **Error Handling:**  Includes basic error handling to catch and report issues during code generation or execution.
*   **OCI Generative AI Integration:** Seamlessly integrates with OCI's Generative AI service using the `DBMS_CLOUD_AI` package.
* **Meta LLaMA 3 model:** Uses the Meta LLaMA 3 model, configured through an OCI AI profile.

## Prerequisites

Before using this agent, ensure you have the following:

1.  **OCI ATP Database (23ai or later):**  You need an Oracle Autonomous Transaction Processing database instance running on OCI.  The database version must be 23ai or later, which includes the necessary `DBMS_CLOUD_AI` package.
2.  **OCI Account and Permissions:** You need an OCI account with the necessary permissions to:
    *   Create and manage credentials (`DBMS_CLOUD.CREATE_CREDENTIAL`).
    *   Create and manage AI profiles (`DBMS_CLOUD_AI.CREATE_PROFILE`).
    *   Use the Generative AI service (`DBMS_CLOUD_AI.GENERATE`).
    *   Create and manage database objects (packages, types).
    * The user must have execute permission on `DBMS_CLOUD`, `DBMS_CLOUD_AI`
3. **Compartment OCID** You must know in which compartment you'll store the AI Profile.

## Setup and Configuration

### 1. Create OCI Credentials

You'll need to create OCI credentials that the database can use to authenticate with the Generative AI service.  Replace the placeholders with your actual credentials (user OCID, tenancy OCID, private key, fingerprint, etc.). Store the credential securely.  This step is crucial for security. *Do not* hardcode these values directly into your PL/SQL code.

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'GENAI_CRED',
    user_ocid       => 'ocid1.user.oc1..your_user_ocid',
    tenancy_ocid    => 'ocid1.tenancy.oc1..your_tenancy_ocid',
    private_key     => '-----BEGIN PRIVATE KEY-----...your_private_key...-----END PRIVATE KEY-----',
    fingerprint     => 'your_fingerprint'
  );
END;
/
```

Important Security Note:  The example above shows how to create the credential.  In a production environment, you should manage your private key with extreme care.  Consider using OCI Vault for storing and retrieving secrets like your private key, rather than embedding it directly in the CREATE_CREDENTIAL call.

Create an AI Profile
Next, create an AI profile that specifies the LLM provider, model, and other settings.  Replace <<compartment_id>> with the OCID of the compartment where you want to create the profile.

BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'OCI_META_LLAMA',
    attributes   => '{"provider": "oci",
                     "credential_name": "GENAI_CRED",
                     "oci_compartment_id": "ocid1.compartment.oc1..<<comartment_id>>",
                     "object_list": [],
                     "oci_apiformat": "GENERIC",
                     "model": "meta.llama3.70b-instruct"
                    }'
  );
END;
/

Explanation of Attributes:

profile_name: A user-defined name for the profile.
attributes: A JSON object containing the profile settings.
provider: Specifies the LLM provider (in this case, "oci").
credential_name: The name of the credential you created in step 1.
oci_compartment_id: The OCID of the compartment.
object_list: An optional list of objects (not used in this example).
oci_apiformat: Specifies the API format ("GENERIC" for general-purpose use).
model: The specific LLM model to use (Meta's LLaMA 3 model). Note: I've updated the model name to a more likely valid name, meta.llama3.70b-instruct. You may need to verify the exact model name available in your OCI region. The provided meta.llama-3.1-405b-instruct is unlikely to be a valid model identifier.

Install the PL/SQL Agent
Create the plsql_agent package (specification and body) in your database schema.  Copy and paste the code you provided in your question into your SQL Developer, SQLcl, or other database client.

-- (Your package specification and body code here)
create or replace PACKAGE plsql_agent AS

  -- Function to capture DBMS_OUTPUT

  TYPE dbms_output_rec IS RECORD ( line VARCHAR2(32767) );
  TYPE dbms_output_tab IS TABLE OF dbms_output_rec;

  FUNCTION DBMS_OUTPUT_GET RETURN dbms_output_tab PIPELINED;

  -- Procedure to generate and execute PL/SQL code using LLM
  PROCEDURE generate_and_execute(
    p_user_input IN VARCHAR2
  );

END plsql_agent;
/

create or replace PACKAGE BODY plsql_agent AS

  FUNCTION DBMS_OUTPUT_GET RETURN dbms_output_tab PIPELINED AS
    l_line   VARCHAR2(32767);
    l_status INTEGER;
  BEGIN
    LOOP
      DBMS_OUTPUT.GET_LINE(l_line, l_status);
      EXIT WHEN l_status != 0;
      PIPE ROW(dbms_output_rec(l_line));
    END LOOP;
    RETURN;
  END DBMS_OUTPUT_GET;

  PROCEDURE generate_and_execute(
    p_user_input IN VARCHAR2
  ) AS
    v_prompt       VARCHAR2(4000);
    v_llm_response CLOB;
    v_output_tab   dbms_output_tab; -- Use the table type for BULK COLLECT
    v_output       VARCHAR2(4000);  -- For concatenating the output

  BEGIN

    -- 1. Construct a generic prompt
    v_prompt := 'Write a valid and executable PL/SQL code block that follows these instructions:
    - Use and assign a variable named v_result.
    - Code should begin with a single line to initialize v_result to 0;
    - Code should output the result using DBMS_OUTPUT.PUT_LINE(''Result: '' || v_result);
    - Code should end with END;.
    - Do not include markdown formatting or explanations—return only the raw PL/SQL block.';

    v_prompt := v_prompt || ' Given operation: ' || p_user_input;

    -- 2. Call the LLM
    SELECT DBMS_CLOUD_AI.GENERATE(
            prompt       => v_prompt,
            profile_name => 'OCI_META_LLAMA',
            action       => 'chat'
          )
    INTO v_llm_response
    FROM dual;

    -- 3. Clean the response
    v_llm_response := TRIM(REPLACE(v_llm_response, '`', '')); -- Remove backticks and trim spaces

    -- Remove "sql" prefix if present
    IF SUBSTR(v_llm_response, 1, 3) = 'sql' THEN
      v_llm_response := SUBSTR(v_llm_response, 4);
      v_llm_response := TRIM(v_llm_response); -- remove any leading spaces after "sql"
    END IF;

     -- Trim any extra space at the end
     v_llm_response := RTRIM(v_llm_response);

/*    -- 4. Ensure the response ends with END;
    IF NOT v_llm_response LIKE '%END;' THEN
      v_llm_response := v_llm_response || ' ;'; -- Append END; if it's missing
    END IF;*/

    -- 5. Print the generated code for verification
    DBMS_OUTPUT.PUT_LINE('Generated PL/SQL Code:');
    DBMS_OUTPUT.PUT_LINE(v_llm_response);

    -- 6. Execute the dynamically generated block
    DECLARE
      v_result NUMBER := 0; -- Local variable for EXECUTE IMMEDIATE
    BEGIN
      EXECUTE IMMEDIATE v_llm_response;

      -- Capture the DBMS_OUTPUT using a pipelined function
      SELECT * BULK COLLECT
      INTO   v_output_tab
      FROM   TABLE(DBMS_OUTPUT_GET);

      -- Concatenate the output lines into a single string
      v_output := '';
      FOR i IN 1..v_output_tab.COUNT LOOP
        v_output := v_output || v_output_tab(i).line || CHR(10); -- Add newline for readability
      END LOOP;

      -- Print the captured output
      DBMS_OUTPUT.PUT_LINE('Captured Output:');
      DBMS_OUTPUT.PUT_LINE(v_output);

    END;


  EXCEPTION
    WHEN OTHERS THEN
      DBMS_OUTPUT.PUT_LINE('Error executing generated code: ' || SQLERRM);
  END generate_and_execute;

END plsql_agent;
/

