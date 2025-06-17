---
draft: false
date: 2025-06-17
categories:
  - Datastack
  - LLM
tags:
  - LLM
  - ollama
  - langgraph
  - homelab
  - datastack
  - pandas
---
# LLM Agents - Local Experiments with LangGraph

## Purpose

This blog documents my personal experiments with building LLM-based agents, running entirely on local hardware.
The main tool used is [LangGraph](https://langgraph.org/), which allows the creation of agents as computation graphs.

This is an ongoing learning project to explore:

* How to design LLM agents.
* How to build data generators leveraging LLM text generation.
* How to handle limitations and control outputs.
<!-- more -->
## Premises

1. Everything runs locally — no cloud dependency — for full control.
2. Focused on building a **data generator agent**.
3. Goal is to study how LLMs can generate structured data (tables) suitable for later use and analysis.
4. Outputs include CSV files for evaluation.
5. The project is in early stages (version 0) and under active development — there may be bugs or unused code.
6. Screenshots and debug prints are included to help track progress.
7. Initial experiments use **Qwen 3 4B** — to understand how smaller SLMs perform locally.

## Objectives

* Understand how to design **LangGraph flows**:

  * Decisions
  * Prompting for entity extraction
  * Capturing intent from LLM responses
* Test whether LLMs can consistently generate structured outputs:

  * Number of columns
  * Number of rows
  * Thematic consistency
  * Controlled specifications
* Explore if the agent can generate **metadata** describing the generated data — critical for making outputs reusable in later stages.
* Implement an **intent node** to decide:

  * Should a new dataset be generated?
  * Should metadata be modified?
  * Can data be reused?
  * This tests LangGraph's ability to handle decision-making flows.
* Final output for each run: **10 example rows** saved as CSV, to evaluate data generation quality.

## Current Status (Version 0)

This is **version 0** — the first prototype. The agent is still evolving.
You will find code samples and links to the GitHub repo containing the current implementation and further updates.

---

## Example CSV Output

Here is a properly formatted CSV with 10 rows of data, ensuring all fields are correctly structured and adhere to the specified requirements.
Each field containing commas or special characters is enclosed in double quotes to maintain CSV integrity.

```
"u12345","john doe","johndoe@example.com","p@ssw0rd!","1990-05-15","male","+1 (212) 555-0198","123 main st, springfield, il, 62704","springfield","illinois","62704","usa","2023-04-05t14:30:45","2023-04-06t10:15:30","active"
"u67890","jane smith","janesmith@domain.com","securepass123!","1985-08-22","female","+44 20 7946 0012","456 oak ave, london, uk, sw1a 1aa","london","uk","sw1a 1aa","uk","2023-04-07t09:45:22","2023-04-08t14:00:15","inactive"
"u34567","emily johnson","emilyj@domain.com","qwerty!123","1995-03-12","other","+33 1 234 5678","789 rue de paris, paris, france, 75001","paris","france","75001","france","2023-04-09t17:20:45","2023-04-10t08:30:10","active"
"u98765","michael brown","michaelb@domain.com","pass123!@#","1988-11-04","male","+1 (310) 555-1234","101 pine st, new york, ny, 10001","new york","new york","10001","usa","2023-04-11t13:15:30","2023-04-12t09:45:20","active"
"u54321","sarah wilson","sarahw@domain.com","secure@123","1992-07-25","female","+44 20 7946 0013","202 brook st, manchester, uk, m1 1aa","manchester","uk","m1 1aa","uk","2023-04-13t10:55:45","2023-04-14t12:10:30","inactive"
"u76543","david miller","davidm@domain.com","test123!@#","1980-02-10","male","+1 (201) 555-3333","303 maple st, chicago, il, 60614","chicago","illinois","60614","usa","2023-04-15t15:30:45","2023-04-16t14:20:15","suspended"
"u23456","laura evans","lauree@domain.com","login!123","1998-09-18","other","+33 1 234 5679","404 rue de lyon, lyon, france, 69001","lyon","france","69001","france","2023-04-17t09:45:20","2023-04-18t12:30:45","active"
"u123456","robert johnson","robertj@domain.com","secure@pass123","1994-01-05","male","+1 (212) 555-9876","505 green st, san francisco, ca, 94101","san francisco","california","94101","usa","2023-04-19t14:00:45","2023-04-20t10:15:30","inactive"
"u654321","jessica williams","jessica@domain.com","test@pass!123","1987-05-05","female","+44 20 7946 0014","606 river st, birmingham, uk, b1 1aa","birmingham","uk","b1 1aa","uk","2023-04-21t13:55:20","2023-04-22t11:45:30","suspended"
"u987654","william davis","williamd@domain.com","pass123!@#","1989-12-25","male","+33 1 234 5670","707 boulevard de paris, paris, france, 75002","paris","france","75002","france","2023-04-23t10:30:45","2023-04-24t14:20:15","active"
```

### Key Notes

* **Phone numbers**: include country codes (e.g., `+1` for the US, `+44` for the UK, `+33` for France).
* **Addresses**: properly formatted with city, state, and ZIP/postcode.
* **Dates**: follow ISO 8601 format (`yyyy-mm-ddThh:mm:ss`).
* **Quotes**: used around fields containing commas or special characters.

This data is ready for import into spreadsheet software or databases.

---

## Observations

* The agent currently tends to **repeat certain patterns**, such as:

  * Similar email addresses with low variability.
  * Some repeated formats, despite temperature set to **1.0**.
* Occasionally, the LLM **injects extra information** not matching the intended structure, causing parsing issues.
* After running multiple times and refining prompts, successful generation of valid CSVs was achieved using **Pandas** for post-processing.
* Improvements are ongoing to better control output variability and consistency.

---

More versions and improvements will be documented in future updates.
