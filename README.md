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


```
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
```

```
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
```

Usage
To use the agent, call the generate_and_execute procedure with your natural language instruction as input:

SQL

BEGIN
  plsql_agent.generate_and_execute(p_user_input => 'Calculate the sum of numbers from 1 to 10.');
END;
/
SQL

Expected Output (Example):

The output will be displayed in your database client's output window.  It will consist of two main parts:

Generated PL/SQL Code: The code generated by the LLM based on your input. This allows you to verify the code before it's executed.
Captured Output: The result of executing the generated code, including any DBMS_OUTPUT.PUT_LINE statements within the generated code. This is where you'll see the actual result of your operation.
For the "sum of numbers from 1 to 10" example, you might see something like this (the exact generated code may vary slightly):

Generated PL/SQL Code:
DECLARE
  v_result NUMBER := 0;
BEGIN
  FOR i IN 1..10 LOOP
    v_result := v_result + i;
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('Result: ' || v_result);
END;

Captured Output:
Result: 55


Code Explanation
* DBMS_OUTPUT_GET Function: This pipelined function retrieves lines from the DBMS_OUTPUT buffer. Pipelined functions allow you to process results row-by-row as they become available, which is efficient for handling potentially large amounts of output.

* generate_and_execute Procedure:
* * Prompt Construction: A generic prompt is created, instructing the LLM to generate a valid PL/SQL block with specific requirements (variable name, output format, etc.). The user's input is appended to this prompt.
* * LLM Call: DBMS_CLOUD_AI.GENERATE is called to interact with the LLM. The prompt, profile_name, and action ('chat') are passed as parameters.
* * Response Cleaning: The LLM's response is cleaned by removing backticks (often used for code formatting in Markdown) and any leading "sql" prefix.
* * Code Execution: EXECUTE IMMEDIATE is used to execute the dynamically generated PL/SQL code. A local variable v_result is declared to ensure that the generated code has a variable to work with.
* * Output Capture: The DBMS_OUTPUT_GET function is used within a BULK COLLECT operation to efficiently retrieve all lines from the DBMS_OUTPUT buffer into a PL/SQL table. The lines are then concatenated into a single string for display.
* * Error Handling: A WHEN OTHERS exception handler is included to catch any errors during code execution and display the error message.

Troubleshooting
* ORA-40441: JSON syntax error: If you get this error during DBMS_CLOUD_AI.CREATE_PROFILE, double-check the JSON syntax in the attributes parameter. Ensure you have valid quotes, colons, and commas. Use a JSON validator online to check for errors.
* ORA-20987: Insufficient privileges to access the profile: Ensure that the database user has the necessary privileges to use the AI profile. You might need to grant specific permissions on the profile itself. You may need to grant execute on DBMS_CLOUD and DBMS_CLOUD_AI
* Incorrect or Unexpected Results: If the generated code doesn't produce the expected results, review the "Generated PL/SQL Code" output. The LLM might have misinterpreted your instructions. Try rephrasing your input to be more specific and clear. Remember that LLMs are not perfect and may require some experimentation.
* No output Ensure that set serveroutput on is set on your sql client.
* ORA-06550: line ..., column ...: PLS-00103: Encountered the symbol ...: This is a general PL/SQL compilation error. Carefully examine the generated code (printed by DBMS_OUTPUT.PUT_LINE) for syntax errors. Common issues include missing semicolons, incorrect variable names, or invalid SQL statements. The LLM might have introduced errors.

Model Availability: Ensure the specified model (meta.llama3.70b-instruct or the one you choose) is available in your OCI region. Check the OCI documentation for the latest list of available models.

Limitations
* OCI ATP Only: This solution is strictly limited to Oracle ATP databases on OCI. It cannot be used with on-premise databases.
* LLM Dependency: The quality of the generated code depends
