# Migration Status Board

> **Last Updated**: [Update this date when modifying the board]  
> **Maintained By**: Core Refactor Team (Gio, Fathul / Luce.sg)

This document tracks the detailed migration status of each module from v1 to v2. Update this board as modules are migrated.

## Quick Status Overview

- **Total Modules**: [Count]
- **Completed**: [Count] ✅
- **In Progress**: [Count] 🔄
- **Not Started**: [Count] ⏳
- **Blocked**: [Count] 🐛

## Status Legend

- ✅ **Complete** - Fully migrated, tested, and in production
- 🔄 **In Progress** - Currently being migrated
- ⏳ **Not Started** - Not yet migrated
- 🐛 **Blocked** - Migration blocked by dependencies or issues
- 🔍 **Review** - Awaiting code review
- ⚠️ **At Risk** - Migration at risk due to issues or delays

## Infrastructure Modules

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Feature Flags | ✅ Complete | N/A | All | - | - | Route-level flags implemented | - |
| Route Integration (Native) | 🔄 In Progress | N/A | Gio, Fathul | - | - | Basic structure in place | - |
| Route Integration (Web) | 🔄 In Progress | N/A | Gio, Fathul | - | - | Basic structure in place | - |
| Build Configuration | ⏳ Not Started | N/A | - | - | - | - | - |
| CI/CD Pipeline | ⏳ Not Started | N/A | - | - | - | - | - |

## Core Feature Modules

### Authentication

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Login | ⏳ Not Started | `login` | - | - | - | - | - |
| Signup | ⏳ Not Started | `signup` | - | - | - | - | - |
| OTP Verification | ⏳ Not Started | `otp` | - | - | - | - | - |
| Password Reset | ⏳ Not Started | `password-reset` | - | - | - | - | - |
| Auth Store | ⏳ Not Started | N/A | - | - | - | - | - |

### Homepage

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Homepage | ⏳ Not Started | `home` | - | - | - | - | - |
| Service List | ⏳ Not Started | N/A | - | - | - | - | - |
| Service Detail | ⏳ Not Started | `service-detail` | - | - | - | - | - |

### Profile

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Profile Overview | ⏳ Not Started | `profile` | - | - | - | - | - |
| Account Info | ⏳ Not Started | `profile/account-info` | - | - | - | - | - |
| Service Profile | ⏳ Not Started | `profile/service-profile` | - | - | - | - | - |
| Addresses | ⏳ Not Started | `profile/addresses` | - | - | - | - | - |
| Contacts | ⏳ Not Started | `profile/contacts` | - | - | - | - | - |
| Payment Methods | ⏳ Not Started | `profile/payment` | - | - | - | - | - |
| Invoices | ⏳ Not Started | `profile/invoices` | - | - | - | - | - |
| Packages | ⏳ Not Started | `profile/packages` | - | - | - | - | - |
| Profile Store | ⏳ Not Started | N/A | - | - | - | - | - |

### Visits

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Visits List | ⏳ Not Started | `visits` | - | - | - | - | - |
| Visit Detail | ⏳ Not Started | `visits/:id` | - | - | - | - | - |
| Reschedule Visit | ⏳ Not Started | `visits/reschedule` | - | - | - | - | - |
| Rate Visit | ⏳ Not Started | `visits/rate` | - | - | - | - | - |
| Visits Store | ⏳ Not Started | N/A | - | - | - | - | - |

### Booking

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Booking Flow | ⏳ Not Started | `booking` | - | - | - | - | - |
| Select Service | ⏳ Not Started | `select-service` | - | - | - | - | - |
| Select Slot | ⏳ Not Started | `booking/select-slot` | - | - | - | - | - |
| Booking Info | ⏳ Not Started | `booking/booking-info` | - | - | - | - | - |
| Booking Confirmation | ⏳ Not Started | `booking/confirmation` | - | - | - | - | - |
| Booking Complete | ⏳ Not Started | `booking/complete` | - | - | - | - | - |
| Booking Store | ⏳ Not Started | N/A | - | - | - | - | - |

### Rewards

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Rewards List | ⏳ Not Started | `rewards` | - | - | - | - | - |
| Promos | ⏳ Not Started | `rewards/promos` | - | - | - | - | - |
| Your Rewards | ⏳ Not Started | `rewards/your-rewards` | - | - | - | - | - |
| Rewards Store | ⏳ Not Started | N/A | - | - | - | - | - |

### Notifications

| Module | Status | Route Flag | Engineers Testing | Started Date | Completed Date | Notes | PR Link |
|--------|--------|------------|-------------------|--------------|----------------|-------|---------|
| Notification List | ⏳ Not Started | `notifications` | - | - | - | - | - |
| Notification Detail | ⏳ Not Started | `notifications/:id` | - | - | - | - | - |
| Notification Settings | ⏳ Not Started | `profile/notification-settings` | - | - | - | - | - |
| Notification Store | ⏳ Not Started | N/A | - | - | - | - | - |

## Shared Modules

These modules are shared between v1 and v2 and do not require migration:

| Module | Status | Notes |
|--------|--------|-------|
| GraphQL Client | ✅ Complete | Shared, no migration needed |
| Monitoring (Sentry, PostHog) | ✅ Complete | Shared, no migration needed |
| Assets | ✅ Complete | Shared, no migration needed |
| Generated GraphQL Types | ✅ Complete | Shared, no migration needed |
| GraphQL Queries | ✅ Complete | Shared, no migration needed |
| Storage Utilities | ✅ Complete | Shared, no migration needed |
| Platform Utilities | ✅ Complete | Shared, no migration needed |

## Migration Timeline

### Q1 2024
- [ ] Infrastructure setup
- [ ] Route integration
- [ ] First feature migration (TBD)

### Q2 2024
- [ ] Core features migration
- [ ] Testing and refinement

### Q3 2024
- [ ] Remaining features migration
- [ ] Final migration steps

### Q4 2024
- [ ] Complete migration
- [ ] v1 code moved to `src/v1/`

## Blockers and Issues

| Module | Issue | Impact | Owner | Status |
|--------|-------|--------|-------|--------|
| - | - | - | - | - |

## Notes

- Update this board weekly during active migration
- Add blockers immediately when discovered
- Link PRs when modules are completed
- Update completion dates when modules are merged to main

## How to Update This Board

1. **Starting Migration**: Change status to 🔄 In Progress, add Started Date, assign Engineers Testing
2. **Completing Migration**: Change status to ✅ Complete, add Completed Date, link PR
3. **Blocked**: Change status to 🐛 Blocked, add entry to Blockers and Issues table
4. **Review**: Change status to 🔍 Review when PR is ready for review

