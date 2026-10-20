---
layout: post
title: "IAM Policy Analyser: Detect Over-Permissions & Security Risks"
date: 2026-05-20
category: projects
summary: "I've built a security tool that analyses AWS IAM policies to identify excessive permissions and potential security risks. It highlights risky configurations and suggests fixes to follow the principle of least privilege."
---



## Objective

The objective of this project was to build a security tool that identifies over-permissive AWS IAM policies and highlights potential privilege-related risks. The goal was to understand how poorly configured permissions can increase the risk of unauthorized access and to help users identify and reduce excessive permissions by following the principle of least privilege.

## Tech Stack / Tools Used

- **Language:** Python
- **Frontend:** Streamlit
- **Backend:** FastAPI
- **Server:** Uvicorn
- **Data Format:** JSON (IAM policy documents)
- **Security Concepts:** AWS IAM, access control, least privilege, permission analysis

## Approach

1. **Policy Input:** Accept AWS IAM policy documents in JSON format.
2. **Policy Analysis:** Parse policy statements and examine actions and resources to identify overly broad permissions.
3. **Risk Detection:** Flag risky configurations, such as wildcard permissions, broad service access, and potentially destructive actions.
4. **Risk Assessment:** Classify findings by severity and calculate a risk score.
5. **Remediation:** Provide findings and suggestions for restricting permissions according to the principle of least privilege.

## Vulnerabilities

The analyzer identifies risky IAM policy configurations, including:

- **Wildcard Actions (`"*"`)** – Allowing unrestricted actions across AWS services.
- **Wildcard Resources (`"*"`)** – Granting permissions across all resources instead of limiting access to specific ones.
- **Full Administrative Access** – Policies that grant broad administrative privileges, increasing the risk of unauthorized or unintended actions.

These findings highlight how excessive permissions can increase the potential impact of compromised credentials or misuse of access.

## Solution 

The analyzer identifies overly permissive IAM policies and provides recommendations to reduce unnecessary access. Wildcard actions can be replaced with specific actions required for the intended task, while wildcard resources can be restricted to relevant AWS resources wherever supported. Policies granting broad administrative privileges can be reviewed and replaced with more narrowly scoped permissions based on actual requirements.

These recommendations help users understand the risks associated with their existing policies and make informed changes to improve access control. By encouraging resource-specific permissions and adherence to the principle of least privilege, the tool helps reduce the potential impact of compromised credentials and unintended actions.

The suggested changes should be reviewed and tested before deployment to ensure that legitimate application and user operations continue to function correctly.

## Challenges Faced

One of the main challenges was designing rules to identify overly permissive IAM policies without flagging every broad permission as the same type of risk. I worked on analyzing policy statements, checking wildcard actions and resources, and identifying configurations that could grant excessive access.

Another challenge was connecting the Streamlit frontend with the FastAPI backend and ensuring that policy data was correctly submitted for analysis and the results were displayed properly. Troubleshooting the API integration helped me better understand how frontend and backend components communicate in a security application.

## Scripts

The project source code includes the policy analysis logic, API endpoints, and user interface for submitting IAM policies and viewing security findings.

- `app.py` – Streamlit interface for uploading or submitting IAM policies and viewing analysis results.
- `main.py` – FastAPI backend for processing policy analysis requests.
- `requirements.txt` – Lists the Python dependencies required to run the project.
- `README.md` – Documents the project setup, usage, and functionality.

**GitHub Repository:** [IAM Policy Analyzer](https://github.com/shuvivyas/IAM-Policy-Analyzer-Detect-Over-Permissions-Security-Risks)

## Results

- Built a tool to analyze AWS IAM policy documents and identify potentially excessive permissions.
- Implemented risk detection for wildcard actions, wildcard resources, and broad administrative access.
- Provided risk assessments and remediation suggestions to help users understand and address insecure permission configurations.
- Applied IAM security concepts, including access control and the principle of least privilege, in a practical security project.
