# SQL Query Chain with LLM

This document explains how to create and utilize an SQL query chain with a Large Language Model (LLM) to generate, correct, and execute SQL queries dynamically.

---

## Overview

We demonstrate a workflow that:

1. Creates an SQL query chain using `create_sql_query_chain`.
2. Displays the contents of the chain for learning purposes.
3. Corrects malformed SQL queries with a custom chain.
4. Executes the final SQL query using the `QuerySQLDataBaseTool`.
5. Uses SQLDatabaseToolkit to dynamically inspect tools and schemas.
6. Implements an agent for interactive SQL query handling.

---

## 1. Setup

### Dependencies:

- Python 3.x
- LLM SDK
- SQL Database connection
- QuerySQLDataBaseTool

### Required Imports

```python
from langchain.chains import chain
from langchain.tools import QuerySQLDataBaseTool
from langchain.schema.runnable import RunnablePassthrough
from langchain.prompts import SystemMessagePromptTemplate, HumanMessagePromptTemplate, ChatPromptTemplate
from langchain.output_parsers import StrOutputParser
from langchain.chains.sql import create_sql_query_chain
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_core.messages import SystemMessage, HumanMessage
from langgraph.prebuilt import create_react_agent
```

---

## 2. Creating SQL Query Chain

```python
execute_query = QuerySQLDataBaseTool(db=db)
sql_query = create_sql_query_chain(llm, db)
```

This initializes the SQL query chain (`sql_query`) using an LLM model and connects it to the database (`db`).

---

## 3. Inspecting SQL Query Chain

To inspect the structure and prompts used inside the SQL query chain:

```python
print(sql_query.get_prompts()[0].pretty_print())
```

This displays the formatted prompt structure inside the chain for debugging and educational purposes.

### Example Output:

```
You are a MySQL expert. Given an input question, first create a syntactically correct MySQL query to run, then look at the results of the query and return the answer to the input question.
Unless the user specifies in the question a specific number of examples to obtain, query for at most 5 results using the LIMIT clause as per MySQL. You can order the results to return the most informative data in the database.
Never query for all columns from a table. You must query only the columns that are needed to answer the question. Wrap each column name in backticks (`) to denote them as delimited identifiers.
Pay attention to use only the column names you can see in the tables below. Be careful to not query for columns that do not exist. Also, pay attention to which column is in which table.
Pay attention to use CURDATE() function to get the current date, if the question involves "today".

Use the following format:

Question: Question here
SQLQuery: SQL Query to run
SQLResult: Result of the SQLQuery
Answer: Final answer here

Only use the following tables:
{table_info}

Question: {input}
```

---

## 4. Additional Function for LLM Interaction

### Defining QnA Chain for Contextual Answers

```python
system = SystemMessagePromptTemplate.from_template("""You are helpful AI assistant who answer user question based on the provided context.""")

prompt = """Answer user question based on the provided context ONLY! If you do not know the answer, just say "I don't know".
            ### Context:
            {context}

            ### Question:
            {question}

            ### Answer:"""

prompt = HumanMessagePromptTemplate.from_template(prompt)

messages = [system, prompt]
template = ChatPromptTemplate(messages)

qna_chain = template | llm | StrOutputParser()

def ask_llm(context, question):
    return qna_chain.invoke({'context': context, 'question': question})
```

This function provides a structured template for asking LLM questions based on a specific context, ensuring the LLM responds only with relevant information.

---

## 5. Correcting Malformed SQL Queries

### Chain Definition:

````python
@chain
def get_correct_sql_query(input):
    context = input['context']
    question = input['question']

    instruction = """
        Use above context to fetch the correct SQL query for following question
        {}

        Do not enclose query in ```sql and do not write preamble and explanation.
        You MUST return only single SQL query.
    """.format(question)

    response = ask_llm(context=context, question=instruction)

    return response
````

This chain:

1. Takes the SQL query context and question as input.
2. The **context** for this chain is the **response** from the `create_sql_query_chain`.
3. Generates a corrected SQL query by instructing the LLM.

### Example Invocation:

```python
response = get_correct_sql_query.invoke({'context': response, 'question': question})
```

---

## 6. Executing SQL Queries

### Defining the Final Execution Chain

```python
final_chain = (
    {'context': sql_query, 'question': RunnablePassthrough()}
    | get_correct_sql_query
    | execute_query
)
```

This pipeline:

1. Generates the SQL query (`sql_query`).
2. Passes the query for correction (`get_correct_sql_query`).
3. Executes the corrected query (`execute_query`).

### Running the Final Chain:

```python
result = final_chain.invoke({'question': question})
print(result)
```

---

## 7. Using SQLDatabaseToolkit

### Initialize Toolkit and Display Tools

```python
toolkit = SQLDatabaseToolkit(db=db, llm=llm)

tools = toolkit.get_tools()
print(tools)
```

This outputs the available tools and their schema for inspecting the database.

---

## 8. Implementing SQL Query Agent

```python
SQL_PREFIX = """You are an agent designed to interact with a SQL database..."""
system_message = SystemMessage(content=SQL_PREFIX)
agent_executor = create_react_agent(llm, tools, state_modifier=system_message, debug=False)

question = "How many employees are there?"
for s in agent_executor.stream({"messages": [HumanMessage(content=question)]}):
    print(s)
    print("----")
```

This creates an interactive agent capable of generating and validating SQL queries step-by-step.

---

## 9. Key Notes

- **Validation**: Ensure the SQL syntax is validated before execution.
- **Security**: Sanitize inputs to avoid SQL injection vulnerabilities.
- **Observability**: Use logs to trace errors and outputs during execution.

---

## 10. Conclusion

This workflow demonstrates how to:

1. Use LLMs to dynamically generate and correct SQL queries.
2. Execute the corrected queries against a database.
3. Inspect prompts and chains for educational insights.
4. Use agents for interactive query validation and execution.

This modular design allows flexibility and scalability for database interaction tasks driven by natural language queries.

