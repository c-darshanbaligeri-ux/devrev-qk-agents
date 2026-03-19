# PRD: Trello – DevRev Airdrop Integration (Example)

> **This is a fictional example** demonstrating the PRD format used by DevRev teams. It follows the same structure, depth, and patterns as real DevRev PRDs.

## Overview

This document outlines the approach to build a two-way sync between Trello and DevRev using the Airdrop platform. The goal is to enable teams using Trello for project management to sync their boards, cards, and members into DevRev for unified search, support workflows, and AI-powered automation.

## Problem Statement

Teams using Trello for project management lose context when customer issues in DevRev relate to work tracked in Trello. Support agents cannot see Trello card status, and product managers manually copy data between systems. This results in ~15 minutes per ticket of context-switching overhead and missed SLA targets due to delayed visibility.

## Goal

1. Enable one-way sync of Trello boards, cards, and members into DevRev
2. Reduce manual data copying between Trello and DevRev
3. Power AI-based search and automation in DevRev using Trello project context
4. Enable DevRev agents to reference Trello card status when resolving tickets
5. Support incremental sync to keep data current

## Persona Queries

| Persona | Query | Intent/Action |
|---------|-------|---------------|
| Support Agent | Show me the Trello card linked to ticket TKT-234 | View related project work for customer issue |
| Product Manager | Which Trello cards have been stuck in "In Review" for more than a week? | Identify bottlenecks affecting customers |
| Engineering Lead | List all Trello cards assigned to me that are linked to P0 tickets | Prioritize work with customer impact |

## Sync Direction

- **Trello → DevRev**: Boards, Lists, Cards, Members, Attachments, Checklists
- **DevRev → Trello**: Not in V1 scope
- **Sync type**: One-way with periodic incremental sync. V2 may add reverse sync.

## Authentication

- **Primary method**: API key + token (Trello uses key/token pair, not OAuth for server-side)
- **Tenant handling**: Each Trello workspace is identified by organization ID
- **Token storage**: Secure keyring storage in DevRev
- **Reference**: https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/

## Object Mapping

| Trello Object | DevRev Object | Sync Direction | Notes |
|---------------|---------------|----------------|-------|
| Board | Part (capability) | → DevRev | Each board becomes a product capability |
| List | Custom field (stage) | → DevRev | List names map to custom stage field on issues |
| Card | Issue | → DevRev | Cards become issues with all metadata |
| Member | Dev User | → DevRev | Matched by email address |
| Checklist | Custom field (checklist) | → DevRev | Checklists stored as structured custom field |
| Attachment | Artifact | → DevRev | Files downloaded and linked to issues |
| Label | Tag | → DevRev | Color labels mapped to DevRev tags |

## Permission Mapping

Not needed for V1. Trello boards are either public, workspace-visible, or private. V1 imports only boards the authenticated user has access to, which implicitly honors permissions. V2 may add workspace-level permission mapping to DevRev groups.

## Proposed Architecture

1. **Authentication**: User provides Trello API key + token through Airdrop settings
2. **Sync unit selection**: User selects which Trello board(s) to import
3. **Schema interpretation**: Snap-in reads board structure (lists, custom fields, labels)
4. **Field mapping**: Auto-suggested mappings with manual override for custom fields
5. **Data extraction**: Cards, members, attachments extracted via Trello REST API

### Mapping Rules
- Board-to-part mapping: Each Trello board → one DevRev capability part
- Card-to-issue mapping: Each card → one DevRev issue
- Field-to-field mapping: Customizable per field
- Label-to-tag mapping: Auto-created if tag doesn't exist
- Lookup linking: Member email → DevRev dev_user email match

## Functional Requirements

- Support secure authentication using Trello API key + token
- Support board-level sync unit selection
- Import cards with all standard fields (title, description, due date, labels, members, checklists)
- Import attachments from cards
- Map Trello list position to a custom "Trello List" field on issues
- Support incremental sync using card modification timestamps
- Ability to search synced Trello data via DevRev AI agents

## Non-Functional Requirements

- **Security**: API tokens stored in DevRev keyrings, never logged or exposed
- **Performance**: Handle boards with up to 10,000 cards. Target: initial sync under 30 minutes for 5,000 cards
- **Rate limits**: Trello API allows 100 requests per 10-second window per token. Extraction must stay within this.
- **Scalability**: Support syncing up to 20 boards per workspace

## Constraints

- Reverse sync (DevRev → Trello) is out of scope for V1
- Trello Power-Up custom fields require paid Trello account — V1 imports only standard fields
- Archived cards are not imported unless explicitly requested
- Trello does not support webhooks for server-side apps without a callback URL — periodic sync only

## Success Criteria

- User can authenticate Trello connection in DevRev
- User can select boards and start import via Airdrop UI
- Cards appear as issues in DevRev with all mapped fields
- Members mapped to DevRev users by email
- Attachments downloadable from imported issues
- Incremental sync imports only changed cards
- Synced data searchable via DevRev AI search

## Release Plan

V1: One-way Trello → DevRev sync (boards, cards, members, attachments, labels). Single release.

## Open Questions

- [ ] Should archived cards be included in initial sync? (Default: no)
- [ ] How to handle cards with no list (orphaned)? (Default: skip)
- [ ] Should board-level settings (background, description) sync to part metadata?

## References

- Trello REST API: https://developer.atlassian.com/cloud/trello/rest/
- Trello authentication: https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/
