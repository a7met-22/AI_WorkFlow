# Agentic Data Analysis: Secure Code-Generation Agent

An experimental **Agentic AI workflow** that translates natural-language
questions into Pandas analysis code, validates the generated code,
executes approved operations against employee data, and attempts
automatic correction when execution fails.

The notebook demonstrates a layered approach to building a tool-using
data-analysis agent around a small language model (**Phi-3.5-mini**),
with an emphasis on controlled execution, schema-aware prompting,
traceability, and resilience to imperfect model output.

> **Project:** Agent Workflow\
> **Notebook:** `Agent_Workflow_.ipynb`\
> **Domain:** Agentic AI · LLMs · Python · Pandas · Data Analysis · Code
> Generation\
> **Environment:** Google Colab

------------------------------------------------------------------------

## 1. Project Overview

The goal is to let users ask analytical questions in **Arabic or
English** and have an AI agent generate and run suitable Pandas
operations on a tabular employee dataset.

Instead of allowing the language model to execute arbitrary Python, the
workflow places programmatic controls between code generation and
execution:

1.  Load the dataset and derive its schema.
2.  Give the model the column names and data types rather than the full
    dataset values.
3.  Generate a proposed analysis or Pandas expression.
4.  Validate the generated code against an explicit set of allowed
    operations.
5.  Execute accepted code against a copy of the DataFrame.
6.  If execution fails for a recoverable reason, ask the model to
    correct the code within a limited number of attempts.
7.  Record decisions and outcomes in a graph for inspection.

The central engineering principle is:

> **A prompt is not a security boundary. The application code must
> enforce the rules before generated code is executed.**

------------------------------------------------------------------------

## 2. Key Features

### Natural-language data analysis

-   Accepts analytical questions in Arabic and English.
-   Uses an LLM to translate user intent into Pandas-based analysis.
-   Supports questions that require filtering, aggregation, grouping,
    and other permitted DataFrame operations.

### Schema-aware context

-   Reads the dataset locally.
-   Builds a schema containing column names and data types.
-   Provides schema context to the model without intentionally including
    the underlying row values in the prompt.

### Code validation

-   Uses Python's Abstract Syntax Tree (**AST**) to inspect generated
    code structurally.
-   Applies allow-list / deny-by-default validation to restrict
    permitted syntax and operations.
-   Checks generated code before it reaches the execution stage.
-   Keeps security-related configuration centralized for easier review
    and maintenance.

### Controlled execution

-   Executes approved analysis against a copy of the DataFrame.
-   Captures printed output with `redirect_stdout`.
-   Keeps the original in-memory DataFrame separate from the working
    copy.

### Self-correction

-   Uses a dedicated correction prompt after eligible execution
    failures.
-   Limits automatic correction to a configurable maximum of **three
    attempts** in the documented workflow.
-   Stops retrying when the failure is classified as an
    environment-related issue or a policy violation.

### Decision traceability

-   Maintains agent state and records actions and outcomes.
-   Uses **NetworkX** to represent decisions and workflow transitions as
    a graph.
-   Supports printing a trace and exporting the graph in GraphML format.

------------------------------------------------------------------------

## 3. Architecture

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Data loading                        Loads the tabular dataset into
                                      Pandas.

  Schema extraction                   Exposes column names and data types
                                      to the model.

  Prompt construction                 Provides instructions, schema
                                      context, and relevant interaction
                                      state.

  LLM generation                      Produces a proposed action or
                                      Pandas code.

  Response parsing                    Extracts the agent's decision and
                                      separates it from user-facing text.

  AST validation                      Checks generated code against the
                                      configured syntax and operation
                                      policy.

  Controlled execution                Runs accepted code on a DataFrame
                                      copy and captures output.

  Self-correction                     Requests a corrected version after
                                      eligible failures, with a bounded
                                      retry count.

  Agent state                         Tracks progress, actions, and
                                      responses.

  Decision graph                      Records workflow decisions and
                                      outcomes for inspection.

  Runner                              Coordinates the complete agent
                                      lifecycle.
  -----------------------------------------------------------------------

### High-level workflow

``` text
User question (Arabic / English)
              |
              v
      Build agent prompt
       + dataset schema
              |
              v
       Language model
        (Phi-3.5-mini)
              |
              v
    Parse generated response
              |
              v
       AST code validation
          /       \
      Rejected   Approved
         |           |
         v           v
   Record outcome  Execute on
                   DataFrame copy
                       |
                 +-----+-----+
                 |           |
               Success     Failure
                 |           |
                 v           v
             Return      Check failure
              result      category
                             |
                     Eligible to retry?
                       /          \
                     Yes           No
                      |             |
                 Self-correction  Stop / report
                 (bounded tries)   failure
                      |
                      v
                 Validate again
                      |
                      v
              Record decision graph
```

------------------------------------------------------------------------

## 4. Technical Highlights

### AST-based validation

The workflow uses Python's `ast` module to inspect the structure of
generated code rather than relying only on text matching.

This is more expressive than a simple blacklist because the validator
can reason about syntax nodes, names, attributes, and calls. The
notebook also uses configured restrictions and permitted operations to
constrain the code-generation surface.

**Important security note:** AST validation is a defense-in-depth
measure, not a complete sandbox. Python code execution through `exec` is
not made secure merely by restricting built-ins or checking an AST.
Production deployments should run generated code in a strongly isolated
environment, such as a dedicated container or restricted worker with
resource limits, no secrets, and no unnecessary network or filesystem
access. The validator itself should also be independently tested against
adversarial inputs.

### Schema-only prompting

The model receives structural information about the dataset, such as
column names and data types, rather than the full set of employee
records.

Benefits: - Reduces unnecessary exposure of raw row-level data to the
model. - Helps the model generate code that uses real column names. -
Keeps data processing under the application's control.

This is a data-minimization measure, not a guarantee of privacy. Column
names and schema metadata may themselves be sensitive in some
environments.

### DataFrame-copy execution

The execution layer works with a copy of the original DataFrame. This
helps prevent ordinary Pandas mutations in one analysis run from
changing the source DataFrame held by the application.

A DataFrame copy is not an operating-system security boundary and does
not prevent arbitrary Python code from accessing other resources.

### Bounded self-correction

A failed generated expression can be sent through a separate correction
workflow. Retry limits prevent an unbounded correction loop and make the
process easier to observe and control.

### Decision graph and observability

The `DecisionGraph` component uses NetworkX to record decisions,
reasons, and outcomes. The graph can be printed for inspection and
exported as GraphML for use in compatible graph tools.

------------------------------------------------------------------------

## 5. Technology Stack

  -----------------------------------------------------------------------
  Technology                          Role
  ----------------------------------- -----------------------------------
  Python                              Core implementation language.

  Google Colab                        Notebook execution environment.

  PyTorch                             Model inference backend.

  Hugging Face Transformers           Model and tokenizer loading;
                                      text-generation pipeline.

  Phi-3.5-mini                        Small language model used for agent
                                      responses and code generation.

  Pandas                              Tabular data loading and analysis.

  Python AST                          Structural inspection and
                                      validation of generated code.

  `dataclasses` / `typing`            State representation and type
                                      annotations.

  `re`                                Response parsing and pattern
                                      matching.

  `io` / `sys`                        Output capture and stream handling.

  NetworkX                            Decision graph construction and
                                      export.

  Google Colab Drive / Files          Dataset access and notebook file
                                      transfer.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 6. Running the Notebook

### Requirements

The notebook is designed for Google Colab and uses a Hugging Face
Transformers model. Depending on the selected model-loading
configuration, it may require a GPU and additional memory.

Before running: - Open the notebook in Google Colab. - Select a runtime
appropriate for the model configuration. - Ensure the dataset file
expected by the notebook is available. - Review the model identifier and
loading settings in the configuration cell. - Run the setup and
model-loading cells before invoking the agent.

### Execution order

Run the notebook cells from top to bottom. The main stages are:

1.  Environment and dependency setup.
2.  Google Drive access and dataset preparation.
3.  Model and tokenizer configuration.
4.  Agent state and response-parsing helpers.
5.  Code-generation agent workflow.
6.  Dataset schema extraction and security configuration.
7.  AST validation and controlled Pandas execution.
8.  Error classification and self-correction.
9.  Decision graph and trace recording.
10. Agent runner and test queries.

### Example interaction

A question such as:

``` text
How does revenue vary across salary over time?
Include one concrete example for each salary group.
```

The agent's intended workflow is to interpret the request, generate an
allowed Pandas analysis, validate it, execute it, and return the
resulting output. The exact result depends on the dataset and the
generated code.

------------------------------------------------------------------------

## 7. Configuration

The notebook centralizes important settings so they can be reviewed and
adjusted without changing the core orchestration logic.

Review the configuration cells for: - Model identifier and loading
options. - Device and precision / quantization settings. - Dataset path
and expected input structure. - Allowed and forbidden syntax or
operations. - Maximum output size. - Maximum self-correction attempts. -
Agent iteration and stopping limits.

Do not commit access tokens, credentials, private datasets, or other
secrets to a public repository. Use environment variables or Colab
secrets for credentials where required.

------------------------------------------------------------------------

## 8. Security Considerations and Limitations

This notebook is an educational prototype, not a production-grade secure
code execution service.

  -----------------------------------------------------------------------
  Area                    Current approach        Important limitation
  ----------------------- ----------------------- -----------------------
  Model context           Schema-focused prompt   Schema information can
                                                  still be sensitive;
                                                  prompts do not enforce
                                                  access control.

  Code checks             AST validation and      Validators can have
                          configured restrictions bypasses and require
                                                  adversarial testing.

  Execution               DataFrame copy and      `exec` remains
                          output capture          powerful; these
                                                  measures do not isolate
                                                  the process.

  Retry behavior          Bounded self-correction A retry can generate
                                                  another unsafe or
                                                  incorrect proposal, so
                                                  every retry must be
                                                  revalidated.

  Output                  Captured standard       Output size, execution
                          output                  time, and memory should
                                                  be constrained in
                                                  production.

  Auditability            Decision graph          A graph trace is useful
                                                  for observability but
                                                  is not, by itself, a
                                                  tamper-proof audit log.
  -----------------------------------------------------------------------

### Recommendations before production use

-   Run generated code in an isolated container or worker process.
-   Apply CPU, memory, execution-time, and output-size limits.
-   Disable network access and restrict filesystem access unless
    explicitly required.
-   Provide only the minimum data and permissions needed for each task.
-   Validate every generated and corrected code candidate.
-   Add adversarial security tests for imports, attribute access,
    indirect calls, and resource exhaustion.
-   Log failures and security rejections without exposing sensitive
    employee data.
-   Review model outputs and analytical results before using them for
    consequential decisions.

------------------------------------------------------------------------

## 9. Repository Structure

``` text
.
├── Agent_Workflow_.ipynb
└── README.md
```

Add any required dataset or configuration files separately, and avoid
publishing private employee data or credentials.

------------------------------------------------------------------------

## 10. Possible Future Improvements

-   Replace in-process `exec` with isolated execution workers.
-   Add unit tests for the AST validator and response parser.
-   Build a labeled evaluation set covering Arabic and English
    analytical questions.
-   Measure code-generation success rate, correction rate, execution
    latency, and answer correctness.
-   Add structured result objects instead of relying only on printed
    output.
-   Add explicit execution budgets and cancellation handling.
-   Persist decision traces with session metadata and timestamps.
-   Compare Phi-3.5-mini with other small and larger models under the
    same evaluation protocol.

------------------------------------------------------------------------

## 11. Key Takeaway

This project demonstrates how an LLM can be integrated into a controlled
data-analysis workflow: the model interprets the request and proposes
code, while the surrounding application is responsible for validation,
execution control, retries, and observability.

The most important design lesson is that **reliable agentic systems
require more than a capable model or a carefully written prompt: they
need explicit application-level controls around every action the model
proposes.**

------------------------------------------------------------------------
