<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [🛠 Engineering Journal: Architecture & Deployment Runbook](#-engineering-journal-architecture--deployment-runbook)
  - [🚨 Issue 1: Race Condition on DB Initialization (data.sql executing before Hibernate schema creation)](#-issue-1-race-condition-on-db-initialization-datasql-executing-before-hibernate-schema-creation)
  - [🚨 Issue 2: Primary Key Generation Mismatch in In-Memory Database (H2)](#-issue-2-primary-key-generation-mismatch-in-in-memory-database-h2)
  - [🚨 Issue 3: SQL Syntax Errors on Special Characters (Python Decorator Seed)](#-issue-3-sql-syntax-errors-on-special-characters-python-decorator-seed)
  - [🚨 Issue 4: Architectural Leakage (DTO acting as Database Entity)](#-issue-4-architectural-leakage-dto-acting-as-database-entity)
  - [🚨 Issue 5: Infrastructure Transition (Migrating Deployment Target to Consolidated App Host)](#-issue-5-infrastructure-transition-migrating-deployment-target-to-consolidated-app-host)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 🛠 Engineering Journal: Architecture & Deployment Runbook

This section documents systemic challenges encountered during the orchestration of our Spring Boot microservices, Docker networking, and local CI/CD runners, along with their permanent resolutions.

Setting up a global documentation repo like this turns a messy troubleshooting session into a production-grade internal corporate wiki.

---

### 🚨 Issue 1: Race Condition on DB Initialization (data.sql executing before Hibernate schema creation)
* **Symptom:** Microservice crashed on startup with a `Table "QUESTION" not found` error during the data seeding phase, even though `ddl-auto=update` or `create` was active.
* **Root Cause:** In Spring Boot 3.x, script-based data initialization (`data.sql`) runs before Hibernate ORM initializes by default.
* **Fix:** Added the following property to defer data execution until after the JPA schema is safely built:
  ```properties
  spring.jpa.defer-datasource-initialization=true

### 🚨 Issue 2: Primary Key Generation Mismatch in In-Memory Database (H2)
Symptom: Application crashed during script execution with NULL not allowed for column "ID".
Root Cause: The Question entity used GenerationType.AUTO. In Hibernate 6, this defaults to a sequence table generator instead of an identity column runner, requiring explicit IDs in manual INSERT statements.
Fix: Refactored the entity primary key to use a direct identity strategy:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private int id;
```

### 🚨 Issue 3: SQL Syntax Errors on Special Characters (Python Decorator Seed)
Symptom: JdbcSQLSyntaxErrorException: Syntax error in SQL statement... on string fields containing contractions (e.g., function's).
Root Cause: Standard SQL (and H2) does not recognize backslash escapes (\') for strings. It treats the backslash as a literal and cuts the string context early.
Fix: Escaped single quotes using standard SQL duplicate single quotes (''):

```sql
'To define a new variable within a function''s scope'
```

### 🚨 Issue 4: Architectural Leakage (DTO acting as Database Entity)
Symptom: Hibernate automatically generated an unwanted question_wrapper table in the database schema.
Root Cause: The QuestionWrapper class was marked with @Entity, turning a data transport object into a persistence object.
Fix: Removed the @Entity definition completely to preserve architectural boundaries. DTOs are now strictly models for API communication.

---

### 🚨 Issue 5: Infrastructure Transition (Migrating Deployment Target to Consolidated App Host)
* **Symptom:** Updates pushed to the microservice repository needed to cleanly target a newly designated, generic application host (`apps-01`) instead of an isolated, service-specific VM alias.
* **Root Cause:** Hardcoded CI/CD variables and local resolution configurations tied deployment execution and testing tools to specific single-purpose hostnames (`question-service-01`).
* **Fix:** Abstracted the network footprint across both local development and remote deployment stages:
  1. Updated the environment-wide `VM_HOST` runner variable under the GitHub Actions secrets configuration panel (`/settings/variables/actions`).
  2. Consolidated local DNS mappings inside the Mac workstation `/etc/hosts` file to resolve the new unified app tier smoothly:
```text
     192.168.237.133   apps-01
     ```

---

### 🚨 Issue 6: GitHub Actions Workflow Failure (Invalid YAML Syntax via Hidden Tab Characters)
* **Symptom:** Pushing pipeline updates resulted in an immediate workflow rejection under GitHub Actions with the error: `Invalid workflow file: You have an error in your yaml syntax on line 44`.
* **Root Cause:** While introducing the deployment `docker pull` sequence, a hidden `Tab` character or irregular indentation block crept into the text block. Because the YAML specification strictly forbids tab characters for nested indentation, the GitHub Actions parser failed to compile the workflow structure.
* **Fix:** Utilized native terminal debugging utilities to audit and correct the formatting locally on the Mac runner before pushing:
  1. Exited standard text mode in `vim` and executed the following commands to instantly expose invisible spacing, layout endings, and tab configurations (`^I` sequences):
     ```vim
     :set list
     :set listchars=tab:▸\ ,trail:·
     ```
  2. Created a local pre-flight automation validation step using Python's native parsing framework to ensure structural integrity prior to standard Git commits:
     ```bash
     python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)" < .github/workflows/deploy.yml
     ```


