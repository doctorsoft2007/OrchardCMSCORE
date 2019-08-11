
## Pre-Alpha

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
- Deferred tasks
- Resource manager
- Placement files
- Background tasks
- Recipes

#### Backlog
- Admin theme
  - Guidelines
  - Tree navigation

## Alpha
__December 2016__

The software is complete enough for internal testing. This is typically done by people other than the software engineers who wrote it, but still within the same organization or community that developed the software.

#### Goals
Have more commonly used modules to start building websites.

#### Fully or partially implemented
- Autoroute
- Tokens
- Deployment (Import/Export)
- Tenants management
- Modules management
- Indexing/Search (missing settings)
- Homepage
- Settings
- Live preview
- Body editors
- Markdown editor
- Menu
- Collapsible admin menu
- RSS API
- XmlRpc/MetaWeblog API
- Scripting (JS)

## Beta
__November 2017__

The software is complete enough for external testing -- that is, by groups outside the organization or community that developed the software. Beta software is usually feature complete, but may have known limitations or bugs. Betas are either closed (private) and limited to a specific set of users, or they can be open to the general public.

#### Fully or partially implemented
- Widgets
- Flows
- Layers
- Projections
- Media Library
  - Media Processing
- Liquid Templating
- Queries (Lucene, SQL)
- Razor Pages
- .NET Standard 2.0
- View localization (PO Files, Pluralization)
- The Agency theme
- Docker images

## Beta 2
__June 2018__

#### Fully or partially implemented
- .NET Core 2.1
  - Precompiled Razor Views
  - Razor Pages
- Workflows
- Forms
- Email
- Password reset
- Social logins
- Azure Blob Storage
- Updated default recipes and themes (Bootstrap 4)
- Data Protection
- Fragment caching

## Beta 3
__April 2019__

#### Fully or partially implemented
- Background tasks management
- GraphQL
- Decoupled CMS
- Taxonomies
- Custom navigation
- Microsoft Account, Google, Twitter, Facebook authentication
- ReCaptcha
- Mini Profiler
- Recipe migrations
- File based Content Definition
- The Coming Soon theme
- Edit and Display modes

## RC
__Setpember 2019__

The software is almost ready for final release. No feature development or enhancement of the software is undertaken; tightly scoped bug fixes are the only code you're allowed to write in this phase, and even then only for the most heinous and debilitating of bugs. 

#### Done
- Content Localization
- RTL admin theme
- Resources CDN
- GitHub authentication
- Facebook applications

#### Backlog
- Localization-neutral fields
- Localization packages (Chinese, Spanish, Arabic, French are the most requested)
- Media CDN support
- Sitemaps
- SEO
- .NET Core 3.0

## 1.0
__October 2019__

## Future releases
- Distributed hosting
- Jobs queue
- Output caching
- Entities API
- GraphQL mutations

## Tracking
- Documentation
- Security
- Accessibility
- Deployment and hosting
- Performance

Source: https://blog.codinghorror.com/alpha-beta-and-sometimes-gamma/
