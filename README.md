# AI Job Hunter

**AI Job Hunter** is a modular AI-powered job application automation system built entirely with **n8n**.

The project automates the job-hunting workflow from **CV processing and job discovery** to **semantic job matching, AI-powered CV tailoring, PDF generation, human approval, application tracking, and email confirmation**.

Instead of building one large workflow, the system is divided into **five independent n8n modules**, controlled by a **Main Orchestrator**.

---

## Features

* Automated CV processing and information extraction
* Multi-source job discovery
* Adzuna and Remotive API integration
* Job normalization, filtering, and deduplication
* Semantic job matching using embeddings
* Hybrid candidate-job relevance scoring
* AI-powered CV tailoring
* AI output fact-checking
* Automatic retry when validation fails
* LaTeX CV generation
* Automatic PDF compilation and verification
* Human-in-the-Loop approval
* Duplicate application prevention
* Airtable application tracking
* Automated email confirmation
* Central n8n workflow orchestration
* Controlled error handling between modules

---

# Architecture

```text
Candidate CV
     |
     v
Module 1
CV Processing
     |
     v
Module 2
Job Search & Filtering
     |
     v
Module 3
Hybrid Job Matching
     |
     v
Module 4
AI CV Tailoring & PDF
     |
     v
Module 5
Application & Tracking
     |
     v
Final Run Report

Controlled by Main Orchestrator
```

---

# Workflow Modules

## Module 1 — CV Processing & Information Extraction

### Objective

Receive the candidate CV, validate the uploaded file, extract readable content, and prepare structured candidate information for the rest of the pipeline.

### Workflow

```text
Trigger
-> Normalize Input
-> Needs Download?
-> File Validation
-> Is File Valid?
-> Route by File Type
-> Text Extraction
-> Text Validation
-> AI Extraction
```

### Key Features

* Supports manual, workflow, and form triggers
* Normalizes incoming CV data
* Supports uploaded files and remote file URLs
* Validates file type and availability
* Routes processing according to file type
* Extracts text from PDFs and text-based documents
* Checks extracted text quality
* Uses Gemini as a PDF fallback when normal extraction is insufficient
* Produces structured candidate information

### Output

```text
Clean + Validated Candidate Profile
```

---

## Module 2 — Job Search, Aggregation & Filtering

### Objective

Search multiple job sources, normalize their responses, remove duplicate or low-quality listings, and produce a clean list of opportunities.

### Job Sources

* Adzuna
* Remotive

### Workflow

```text
Normalize Search Config
        |
        +--> Build Adzuna Query
        |        |
        |        v
        |    Adzuna API
        |        |
        |        v
        |    Normalize Adzuna
        |
        +--> Build Remotive Query
                 |
                 v
             Remotive API
                 |
                 v
          Normalize Remotive
                 |
                 v
           Merge Sources
                 |
                 v
           Flatten Jobs
                 |
                 v
        Description Mining
                 |
                 v
          Deduplication
                 |
                 v
        Quality Filtering
                 |
                 v
          Assemble Jobs
```

Because Adzuna and Remotive return different JSON structures, each source is normalized before the results are merged.

### Output

```text
Normalized + Deduplicated + Filtered Job List
```

---

## Module 3 — Hybrid Job Matching

### Objective

Compare the candidate profile against discovered job opportunities and calculate relevance scores.

The system goes beyond traditional keyword matching by using **semantic embeddings**.

### Workflow

```text
Normalize Input
      |
      v
Build Embedding Batch Request
      |
      v
Get Batch Embeddings
      |
      v
Compute Hybrid Scores
      |
      v
Ranked Jobs
```

### Matching Strategy

```text
Candidate Profile
       |
       v
   Embeddings
       |
       +--------------------+
       |                    |
       v                    v
Semantic Similarity   Other Matching Signals
       |                    |
       +---------+----------+
                 |
                 v
          Hybrid Score
                 |
                 v
           Ranked Jobs
```

This allows the system to recognize relevant opportunities even when the wording in a CV differs from the wording used in a job description.

### Output

```text
Ranked Job Opportunities + Hybrid Scores
```

---

## Module 4 — AI CV Tailoring & Document Generation

### Objective

Tailor the candidate's CV for a selected job while ensuring that the AI does not introduce unsupported information.

### Workflow

```text
Normalize Input
      |
      v
LLM Tailor — Attempt 1
      |
      v
Parse & Fact-Check
      |
      v
Fact-Check Passed?
      |
   +--+--+
   |     |
  YES    NO
   |     |
   |     v
   |  Prepare Regeneration
   |     |
   |     v
   |  LLM Tailor — Retry
   |     |
   |     v
   |  Parse & Fact-Check
   |
   v
Reassemble Tailored LaTeX
      |
      v
Compile LaTeX to PDF
      |
      v
Verify Compiled PDF
      |
      v
Assemble Application Package
      |
      v
Create PDF Binary
```

### AI Fact-Checking

The system validates AI-generated CV content before generating the final document.

If the first AI response fails validation, the system automatically attempts regeneration.

If the retry still fails, the workflow emits a controlled failure instead of producing an inaccurate CV.

### Document Generation

```text
Validated CV Content
        |
        v
Tailored LaTeX
        |
        v
PDF Compilation
        |
        v
PDF Verification
        |
        v
Final Application Package
```

### Output

The application package contains the candidate information, selected job, tailored CV, generated PDF, and the information required by the next module.

---

## Module 5 — Application & Tracking

### Objective

Validate the final application package, prevent duplicate applications, obtain human approval, perform the submission stage, record the application, and send confirmation.

### Workflow

```text
Normalize Input
      |
      v
Validate Package
      |
      v
Package Valid?
      |
      v
Duplicate Check — Airtable
      |
      v
Is Duplicate?
      |
      v
Wait for Human Approval
      |
      v
Process Approval Decision
      |
      v
Approved?
      |
      v
Mock Submission
      |
      v
Validate Submission
      |
      v
Persist Tracking Record
      |
      v
Prepare Email Attachments
      |
      v
Send Confirmation Email
      |
      v
Submitted
```

### Human-in-the-Loop

The system includes a human approval checkpoint before the application submission stage.

```text
Application Ready
       |
       v
Wait for Human Approval
       |
   +---+---+
   |       |
APPROVE  REJECT
   |       |
   v       v
Continue  Stop
```

This keeps the candidate in control of the final application decision.

### Duplicate Application Protection

Before continuing with an application, the system searches Airtable using the candidate and job identifiers.

This prevents repeated applications to the same opportunity.

### Application Tracking

Successful application information is stored in **Airtable**, providing a persistent record for application tracking and future duplicate detection.

### Email Confirmation

After the submission and tracking stages are completed, the workflow prepares the relevant application attachments and sends an automated confirmation email to the candidate.

> **Current Submission Status:** The submission stage currently uses a **Mock Submission** for development and testing. Real ATS or supported job-platform integrations can be added as future extensions.

---

# Main Orchestrator

The **Main Orchestrator** controls the complete n8n system.

Rather than placing every node inside one massive workflow, the application is separated into independent modules.

### Orchestrator Workflow

```text
Normalize Orchestrator Input
        |
        v
Execute Module 1
        |
        v
Succeeded?
        |
        v
Execute Module 2
        |
        v
Succeeded?
        |
        v
Execute Module 3
        |
        v
Succeeded?
        |
        v
Execute Module 4
        |
        v
Succeeded?
        |
        v
Execute Module 5
        |
        v
Final Run Report
```

Every module is checked before the next module is allowed to execute.

If a module fails, the orchestrator stops the pipeline safely and reports the failed stage.

---

# Complete End-to-End Flow

```text
Candidate Uploads CV
          |
          v
     CV Processing
          |
          v
 Candidate Extraction
          |
          v
      Job Search
          |
          v
Job Source Normalization
          |
          v
     Deduplication
          |
          v
    Quality Filtering
          |
          v
 Semantic Embeddings
          |
          v
 Hybrid Job Matching
          |
          v
     Job Ranking
          |
          v
     CV Tailoring
          |
          v
     Fact Checking
          |
          v
    PDF Generation
          |
          v
   Package Validation
          |
          v
    Duplicate Check
          |
          v
    Human Approval
          |
          v
   Submission Stage
          |
          v
 Airtable Tracking
          |
          v
 Email Confirmation
          |
          v
    Final Run Report
```

---

# Technology Stack

| Technology                  | Role                                                        |
| --------------------------- | ----------------------------------------------------------- |
| **n8n**                     | Core automation platform, workflow logic, and orchestration |
| **AI / LLM APIs**           | CV extraction and tailoring                                 |
| **Embeddings**              | Semantic candidate-job matching                             |
| **Gemini**                  | PDF extraction fallback                                     |
| **Adzuna API**              | Job discovery                                               |
| **Remotive API**            | Remote job discovery                                        |
| **Airtable**                | Application tracking and duplicate detection                |
| **LaTeX**                   | Tailored CV document generation                             |
| **PDF Compilation Service** | Final PDF generation                                        |
| **Gmail / Email**           | Candidate confirmation                                      |

---

# Repository Structure

```text
AI-Job-Hunter/
|
├── workflows/
│   ├── 01-CV-Processing.json
│   ├── 02-Job-Search.json
│   ├── 03-Hybrid-Job-Matching.json
│   ├── 04-AI-CV-Tailoring.json
│   ├── 05-Application-Tracking.json
│   └── Main-Orchestrator.json
|
├── screenshots/
│   ├── module-1.png
│   ├── module-2.png
│   ├── module-3.png
│   ├── module-4.png
│   ├── module-5.png
│   └── main-orchestrator.png
|
├── docs/
│   └── technical-report.md
|
├── .gitignore
├── LICENSE
└── README.md
```

---

# Setup

## 1. Access n8n

The project is built entirely using **n8n workflows**.

You can use:

* n8n Cloud
* A self-hosted n8n instance

## 2. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/AI-Job-Hunter.git
cd AI-Job-Hunter
```

## 3. Import the Workflows

Open your n8n instance and import the workflow JSON files from the `/workflows` directory.

Import the five modules and the Main Orchestrator.

## 4. Configure Credentials

Configure the required credentials directly inside your n8n instance.

Depending on your configuration, these may include:

```text
AI / LLM Provider
Embedding Provider
Gemini
Adzuna
Airtable
Gmail / Email
PDF Compilation Service
```

## 5. Configure the Main Orchestrator

After importing the workflows, verify that each **Execute Workflow** node inside the Main Orchestrator points to the correct imported module.

Then configure the required input parameters and execute the orchestrator.

---

# Security

**Never commit API keys, access tokens, passwords, or credentials to GitHub.**

n8n credentials should be configured through the n8n credential system.

Before publishing exported workflow JSON files, inspect nodes such as:

```text
HTTP Request
Code
Set / Edit Fields
Headers
Query Parameters
Webhook URLs
```

Make sure no private API key, token, email address, personal CV data, or other sensitive information has been hard-coded into the workflow.

---

# Design Principles

### Modular Architecture

Each workflow performs one clear responsibility, making the project easier to maintain, test, debug, extend, and reuse.

### Human Control

AI assists with the application process, but a human approval checkpoint remains before submission.

### Semantic Matching

Embeddings provide deeper candidate-job matching than simple keyword comparison.

### AI Validation

AI-generated CV content is fact-checked before document generation.

### Controlled Failure

Validation and success gates prevent invalid data from propagating through the pipeline.

### Fallback Processing

Gemini provides an alternative extraction path when normal PDF extraction is insufficient.

### Application Deduplication

Airtable tracking prevents repeated applications to the same job.

---

# Current Project Status

| Feature                   | Status                    |
| ------------------------- | ------------------------- |
| CV Processing             | Complete                  |
| CV Information Extraction | Complete                  |
| Adzuna Job Search         | Complete                  |
| Remotive Job Search       | Complete                  |
| Job Normalization         | Complete                  |
| Deduplication             | Complete                  |
| Quality Filtering         | Complete                  |
| Semantic Embeddings       | Complete                  |
| Hybrid Matching           | Complete                  |
| AI CV Tailoring           | Complete                  |
| AI Fact Checking          | Complete                  |
| Automatic AI Retry        | Complete                  |
| LaTeX Generation          | Complete                  |
| PDF Compilation           | Complete                  |
| PDF Verification          | Complete                  |
| Human Approval            | Complete                  |
| Airtable Tracking         | Complete                  |
| Duplicate Detection       | Complete                  |
| Email Confirmation        | Complete                  |
| Main Orchestrator         | Complete                  |
| Real Job Submission       | Mock / Future Integration |

---

# Future Improvements

Potential future improvements include:

* Real ATS integrations
* Additional job-search APIs
* LinkedIn job discovery where supported
* Automatic cover-letter generation
* Advanced salary filtering
* Location preference scoring
* Candidate preference learning
* Application analytics
* Interview tracking
* Recruiter follow-up workflows
* Additional database integrations
* Vector database support
* Candidate dashboard

---

# Project Goal

AI Job Hunter is designed as a **controlled, intelligent, and modular job-hunting automation pipeline**.

It combines:

```text
n8n Automation
       +
Job Discovery
       +
Semantic Matching
       +
Generative AI
       +
Fact Checking
       +
Human Approval
       +
Application Tracking
```

The goal is to make the job application process more targeted, traceable, and reliable while maintaining human control over the final submission decision.

---

# Disclaimer

This project is intended for educational, research, and workflow automation purposes.

Users are responsible for complying with the terms of service and API policies of any job platforms or external services connected to the workflows.

Automated submission should only be enabled for services that permit this type of integration.

---

# License

This project is available under the license specified in the repository.

Contributions, feedback, and suggestions are welcome.
