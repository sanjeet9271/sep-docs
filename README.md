# SEP Documentation

This repository contains database schema documentation for the SEP (Salesforce Event Portal) system.

## Contents

The documentation is organized in the following directory:

### `cases_table_schema/`

Contains comprehensive schema documentation for the cases management system:

- **`cases.md`** - Main cases table schema, including database structure, API mappings, and SOQL queries
- **`attachments.md`** - Case attachments schema and Salesforce field mappings
- **`comments.md`** - Case comments schema and related documentation

## Overview

These schemas define the PostgreSQL database structure for syncing and managing Salesforce cases, including:

- Case records with warranty claims, RMA repairs, and other case types
- File attachments associated with cases
- Comments and notes on cases

Each documentation file includes:
- Database table definitions (PostgreSQL)
- Column mappings to Salesforce fields
- Indexes and constraints
- SOQL query examples (where applicable)

## Usage

Browse the markdown files in `cases_table_schema/` to understand the database structure and API mappings for the SEP system.

