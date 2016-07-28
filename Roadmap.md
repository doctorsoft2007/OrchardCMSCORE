
## Pre-Alpha (Current)

The software is still under active development and not feature complete or ready for consumption by anyone other than software developers. There may be milestones during the pre-alpha which deliver specific sets of functionality, and nightly builds for other developers or users who are comfortable living on the absolute bleeding edge.

#### Goals
Validate all the core components of Orchard work together to enable module development. We should be able to setup an Orchard site and run a blog engine with a minimal set of custom modules.

#### Fully or partially implemented
- Setup
- Multi-tenancy
- Theming
  - Shapes
- Document database with migrations
- Declarative dependency injection
- Navigation API
- Event bus
- Authentication & Authorization
- Content Items API
  - Content Parts and Content Fields
- Content Types module
  - Managing content types and settings
- List module
- Modules support
  - Dynamic compilation and loading
- Deferred tasks
- Resource manager
- Placement files
- Background tasks

#### Backlog
- Recipes
- Admin theme
  - Guidelines
  - Tree navigation

## Alpha

The software is complete enough for internal testing. This is typically done by people other than the software engineers who wrote it, but still within the same organization or community that developed the software.

#### Goals
Have more commonly used modules to start building websites.

#### Backlog
- Localization
- Import/Export
- Tenants management
- Indexing/Search
- Alias/Autoroute
- Tokens
- Default theme

## Beta

The software is complete enough for external testing -- that is, by groups outside the organization or community that developed the software. Beta software is usually feature complete, but may have known limitations or bugs. Betas are either closed (private) and limited to a specific set of users, or they can be open to the general public.

#### Backlog
- Layouts
- Widgets
- Scripting (JS)
- Taxonomies
- Media Library
- Media processing
- Output caching
- Glimpse

## RC

The software is almost ready for final release. No feature development or enhancement of the software is undertaken; tightly scoped bug fixes are the only code you're allowed to write in this phase, and even then only for the most heinous and debilitating of bugs. 

#### Backlog
- Dynamic forms
- Projections
- Jobs queue
- Workflows
- Email
- Redis
- Message bus

## Tracking
- Security
- Deployment and hosting (Azure, Linux, ...)
- Performance


Source: https://blog.codinghorror.com/alpha-beta-and-sometimes-gamma/
