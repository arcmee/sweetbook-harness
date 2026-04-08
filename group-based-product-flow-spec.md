# Group-Based Product Flow Spec

## Purpose

This document captures the current product planning decisions for the SweetBook group-based flow.
It is intended to guide future UI, backend, and workflow implementation.

## Core Product Principle

- The service is group-based.
- A group is the main unit that owns members, events, voting, photo selection, and ordering.
- Likes are only a priority signal.
- Final photo selection authority belongs to the group owner.

## User Roles

### Group Owner

- Can manage the group
- Can invite members
- Can transfer owner authority
- Cannot leave the group while still being the owner
- Can extend the voting deadline
- Can end voting early
- Can enter the final photo selection page after voting ends
- Can complete the book order

### Group Member

- Can upload photos to an event
- Can like photos during the voting period

## Global UI

### Top Right Account Area

The top-right account UI must provide:

- account information
- notifications
- list of groups the user belongs to
- password change entry

### Notifications

At minimum, notifications must include:

- events where the user has not voted yet
- incoming group invitations

### My Groups List

- The account area must show the list of groups the user belongs to.
- Clicking a group in this list must move the user to that group's page.

## Main Page

The main page should work as a participation-focused dashboard.

- Show currently active voting events
- Group those events by group
- For each event, show up to 5 preview photos with the most likes

## Group List -> Group Page

- Clicking a group must open that group's page.

### Group Page Contents

- group-level hub page
- event list
- member list
- leave group button
- member invitation popup

### Event List on Group Page

Each event item should show:

- event name
- short description
- voting period

Clicking an event in this list must move the user to that event's page.

### Member List on Group Page

The member area must support:

- showing current members
- owner transfer

### Group Invitation

- Invitation is triggered from the group page
- UI form should be implemented as a popup
- Members are searched by member ID

### Leave Group Rule

- Members can leave a group
- The group owner cannot leave directly
- Owner transfer is required before owner exit

## Event Page

The event page is the main collaboration area for a single event.

### Event Creation Input

When creating an event, the following fields are required:

- event name
- short description
- voting period

### Event Page Features

- Members can upload photos for the event
- Uploaded photos must be shown in a grid
- Users can click photos to like them
- Each photo must display its like count

## Voting Rules

- Each event has a voting period
- Likes are only available during that period
- After voting ends, likes are no longer allowed
- The owner can extend the deadline
- The owner can end voting early

Voting ends when either of the following happens:

- the configured deadline passes
- the owner manually closes voting

## Selection Phase

- The final selection page is only available after voting ends
- Only the group owner can enter the final selection page
- Likes should be used as recommendation signals, not as the final decision

### Selection Page Goal

The selection page should be shaped so it can easily simulate the SweetBook book-building UI.

This means:

- it should feel like a real book preparation workspace
- it should support final owner-driven selection
- it should be easy to connect later to the SweetBook order handoff flow

## Order Phase

After final photo selection, the owner should be able to move into ordering.

### Order UI Requirements

- show a simple payment window
- reflect SweetBook pricing
- allow selection of book quantity
- keep the payment UI simple for now
- allow the order to be finalized

## End-to-End User Flow

1. User logs in
2. User sees active voting events on the main page, grouped by group
3. User enters a group page
4. User reviews events and members on the group page
5. User enters an event page
6. Members upload event-related photos
7. Members like photos during the voting period
8. Owner extends or closes voting if needed
9. After voting closes, owner enters the final photo selection page
10. Owner selects final photos
11. Owner enters the order page
12. Owner chooses quantity, reviews price, and completes the order

## State Model

Suggested event states:

- `draft`
- `voting`
- `voting_closed`
- `selecting`
- `ready_to_order`
- `ordered`

## Summary of Key Business Rules

- The service is group-based
- Events belong to groups
- Voting is time-bound
- Likes are recommendation signals only
- Final selection belongs to the group owner
- The owner can extend or close voting
- Ordering starts only after voting is closed and owner selection is complete
