<!--
Sync Impact Report
- Version change: template → 1.0.0 (initial adoption)
- Modified principles: template placeholders → I. Simple, Readable Code; II. Session-Bound Patient Identity; III. Atomic Appointment Booking; IV. Safe Clinical Boundaries; V. Explicit Guardrails
- Added sections: Clinical Safety Constraints; Development Workflow
- Removed sections: none
- Follow-up TODOs: original ratification date is unknown
-->
# Clinic Desk Constitution

## Core Principles

### I. Simple, Readable Code
Implement the smallest clear solution that meets the requirement. Names and control flow MUST be easy to follow. Comments MUST be concise, single-line, and explain why non-obvious behavior exists; they MUST NOT restate the code.

### II. Session-Bound Patient Identity
The authenticated application session is the sole source of patient identity. Patient identity MUST NOT be accepted from AI output, model-generated text, or untrusted request fields. The server MUST authorize every patient-specific read or write against the logged-in session.

### III. Atomic Appointment Booking
The system MUST prevent overlapping bookings for the same appointment slot, including when requests arrive simultaneously. Booking checks and reservation MUST be enforced atomically by the authoritative persistence layer; application-level checks alone are insufficient.

### IV. Safe Clinical Boundaries
The system MUST NOT diagnose, recommend medicines, or provide treatment advice. When a user reports or appears to describe an emergency, the response MUST direct them to call Pakistan emergency service 1122 and seek immediate help.

### V. Explicit Guardrails
Every AI-assisted flow MUST validate inputs and outputs, enforce authorization and clinical boundaries outside the model, and fail safely when validation or required services fail. AI output MUST be treated as untrusted data and MUST NOT directly authorize actions or modify protected records.

## Clinical Safety Constraints

Patient identity, appointment integrity, and clinical safety requirements in this constitution apply to every interface, integration, and background job. A feature that cannot meet these constraints MUST NOT be released.

## Development Workflow

Changes MUST preserve these principles and include appropriate automated checks for identity authorization, concurrent booking, and clinical guardrails when those behaviors are affected. Reviewers MUST verify the checks and any changed user-facing safety language before release.

## Governance

This constitution is the project's governing standard for product behavior and engineering decisions. Amendments MUST be made through review, document the reason and impact, and update this version and the last-amended date. Use semantic versioning: MAJOR for incompatible removals or redefinitions, MINOR for new or materially expanded principles or sections, and PATCH for clarifications that do not change requirements. Every change MUST be reviewed for compliance with all principles; any exception MUST be documented with its rationale and safeguards. Feature-specific intents MUST proceed through the applicable Spec Kit workflow.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date is not recorded | **Last Amended**: 2026-10-03
