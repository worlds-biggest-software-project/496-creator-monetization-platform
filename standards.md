# Standards & API Reference

> Project: Creator Monetization Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

- **ISO 27001 — Information Security Management Systems**
  https://www.iso.org/standard/27001
  Provides requirements for establishing, implementing, and maintaining an information security management system. Essential for any creator platform handling subscriber payment data and personal information.

- **ISO 27701 — Privacy Information Management**
  https://www.iso.org/standard/71670.html
  Extension to ISO 27001 covering privacy management. Relevant to GDPR and CCPA compliance for platforms processing subscriber data across jurisdictions.

- **ISO 8583 — Financial Transaction Card Messages**
  https://www.iso.org/standard/31628.html
  Standard for interchange message specifications for electronic transactions. Relevant to understanding the payment processing layer that underpins creator platform transactions.

### W3C & IETF Standards

- **RFC 6749 — OAuth 2.0 Authorization Framework**
  https://datatracker.ietf.org/doc/html/rfc6749
  Core authorisation framework used by all major creator platforms for third-party API access and user authentication flows.

- **RFC 6750 — OAuth 2.0 Bearer Token Usage**
  https://datatracker.ietf.org/doc/html/rfc6750
  Defines how to use bearer tokens in HTTP requests to access OAuth 2.0 protected resources. Required for API authentication.

- **OpenID Connect Core 1.0**
  https://openid.net/specs/openid-connect-core-1_0.html
  Identity layer on top of OAuth 2.0 enabling user authentication and Single Sign-On. Used for social login (Google, Apple, etc.) on creator platforms.

- **RFC 4287 — The Atom Syndication Format**
  https://datatracker.ietf.org/doc/html/rfc4287
  IETF standard for web content syndication. Used alongside RSS for podcast and newsletter feed distribution.

- **RSS 2.0 Specification**
  https://www.rssboard.org/rss-specification
  De-facto standard for content syndication feeds. Podcast distribution relies on RSS 2.0 extended with iTunes and Podcast namespace tags.

- **PSP-1 Podcast RSS Specification**
  https://github.com/Podcast-Standards-Project/PSP-1-Podcast-RSS-Specification
  Community standard for podcast RSS feeds combining RSS 2.0, iTunes, Podcast, and Atom namespace extensions.

- **RFC 7231 — HTTP/1.1 Semantics and Content**
  https://datatracker.ietf.org/doc/html/rfc7231
  Defines HTTP methods, status codes, and content negotiation. Foundation for all REST API implementations.

- **RFC 8288 — Web Linking**
  https://datatracker.ietf.org/doc/html/rfc8288
  Defines a model for relationships between web resources. Used for pagination and resource discovery in REST APIs.

### Data Model & API Specifications

- **OpenAPI Specification 3.1**
  https://spec.openapis.org/oas/v3.1.0
  Standard for describing RESTful APIs. Enables auto-generation of documentation, client SDKs, and server stubs. Most creator platform APIs follow REST conventions that can be documented with OpenAPI.

- **JSON Schema 2020-12**
  https://json-schema.org/specification
  Vocabulary for validating JSON data structures. Used for API request/response validation, configuration schemas, and data contract enforcement.

- **GraphQL Specification**
  https://spec.graphql.org/
  Query language for APIs enabling clients to request exactly the data they need. Alternative to REST for creator platform APIs, particularly useful for complex dashboard data queries.

- **Standard Webhooks Specification**
  https://www.standardwebhooks.com/
  Open standard for webhook delivery providing consistent signing, verification, and retry semantics. Adopted by platforms like Render and Polar for reliable event notification delivery.

- **Model Context Protocol (MCP)**
  https://modelcontextprotocol.io/specification/2025-11-25
  Open protocol enabling integration between LLM applications and external data sources. Relevant for building AI-native creator tools that connect to platform data. Official SDKs available for TypeScript, Python, C#, Java, and Swift. Over 500 public MCP servers as of 2026.

### Security & Authentication Standards

- **PCI DSS v4.1 — Payment Card Industry Data Security Standard**
  https://www.pcisecuritystandards.org/standards/pci-dss/
  Global standard for entities storing, processing, or transmitting cardholder data. Mandatory for any creator platform handling payments. Version 4.1 emphasises continuous monitoring, stronger authentication, and risk-based security controls.

- **OWASP API Security Top 10 (2023)**
  https://owasp.org/API-Security/
  Industry-standard list of critical API security risks including Broken Object Level Authorization, Broken Authentication, and Unrestricted Resource Consumption. Essential reference for securing creator platform APIs.

- **GDPR — General Data Protection Regulation**
  https://gdpr.eu/
  EU regulation governing the processing of personal data. Requires explicit consent, data portability, right to deletion, and breach notification for platforms serving EU subscribers.

- **CCPA/CPRA — California Consumer Privacy Act**
  https://oag.ca.gov/privacy/ccpa
  California privacy law requiring disclosure of data collection practices, opt-out mechanisms, and data deletion rights. 2026 updates include mandatory cybersecurity audits and automated decision-making risk assessments.

- **COPPA — Children's Online Privacy Protection Act**
  https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa
  US federal law regulating collection of personal information from children under 13. Creator platforms must implement age-gating when content may reach younger audiences.

- **CAN-SPAM Act**
  https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
  US law governing commercial email. Requires unsubscribe mechanisms, honest subject lines, and physical address inclusion in all marketing emails sent by creator platforms.

### MCP Server Specifications

The Model Context Protocol is directly relevant to building AI-native features for a creator monetisation platform. An MCP server could expose creator analytics, subscriber data, and content performance metrics to AI assistants for automated insights and recommendations.

- **MCP Specification (November 2025)**
  https://modelcontextprotocol.io/specification/2025-11-25
  Authoritative protocol requirements based on the TypeScript schema.

- **MCP GitHub Repository**
  https://github.com/modelcontextprotocol/modelcontextprotocol
  Source for specification, SDKs, and reference implementations.

- **2026 MCP Roadmap**
  https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
  Planned enhancements including improved agent workflow support and governance processes.

## Similar Products — Developer Documentation & APIs

### Patreon
- **Description:** Membership platform enabling creators to earn recurring revenue through tiered subscriptions. Over 8 million monthly paying patrons across 180+ countries.
- **API Documentation:** https://docs.patreon.com/
- **SDKs/Libraries:** Community libraries available; official SDK support limited
- **Developer Guide:** https://www.patreon.com/portal/start/quick-start
- **Developer Community:** https://www.patreondevelopers.com/
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0 with application registration required

### Kajabi
- **Description:** All-in-one platform for courses, memberships, email marketing, community, coaching, and podcasts. AI-assisted content creation and marketing automation.
- **API Documentation:** https://developer.kajabi.com/docs/
- **SDKs/Libraries:** Python, Java, Ruby SDKs
- **Developer Guide:** https://help.kajabi.com/en/articles/12696419-using-kajabi-s-public-api
- **Standards:** RESTful API, JSON
- **Authentication:** API Key + API Secret (Settings > Account Details > API Credentials). API available on Pro plan or $25/mo add-on.

### Gumroad
- **Description:** Digital product sales platform for ebooks, courses, software, templates, and subscriptions. 10% flat transaction fee with no monthly costs.
- **API Documentation:** https://app.gumroad.com/api
- **SDKs/Libraries:** Node.js (gumroad-api on npm), Elixir (gumroad_elixir on Hex)
- **Developer Guide:** https://gumroad.com/features
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0 with Application ID and Secret

### Ko-fi
- **Description:** Tipping, memberships, commissions, and digital product sales with zero monthly fees. Direct payouts through creator's own PayPal or Stripe account.
- **API Documentation:** https://help.ko-fi.com/hc/en-us/articles/360004162298-Does-Ko-fi-have-an-API-or-webhook
- **SDKs/Libraries:** No official SDKs; community PHP implementations on GitHub
- **Developer Guide:** N/A — limited to webhook setup documentation
- **Standards:** Webhook (HTTP POST), JSON payload
- **Authentication:** Webhook verification token

### Substack
- **Description:** Newsletter and podcast publishing platform with paid subscriptions, Notes social feed, and Substack TV. Built-in audience discovery engine.
- **API Documentation:** https://support.substack.com/hc/en-us/articles/45099095296916-Substack-Developer-API (official — limited to LinkedIn profile lookup)
- **SDKs/Libraries:** Unofficial: substack-api (Python, PyPI), TypeScript community library
- **Developer Guide:** https://substack-api.readthedocs.io/ (unofficial)
- **Standards:** RSS/Atom for content feeds; no formal REST API standard
- **Authentication:** Application-based access with Terms of Use agreement (official); no authentication for public RSS feeds

### Fourthwall
- **Description:** Creator storefront with print-on-demand merchandise, digital products, memberships, and deep social platform integrations (TikTok Shop, YouTube Shopping, Twitch).
- **API Documentation:** https://docs.fourthwall.com/storefront-api/
- **SDKs/Libraries:** Vercel Commerce template on GitHub (github.com/FourthwallHQ/vercel-commerce)
- **Developer Guide:** https://docs.fourthwall.com/storefront/overview
- **Standards:** REST/JSON, Storefront API with token authentication
- **Authentication:** Storefront token (Settings > For Developers)

### Uscreen
- **Description:** Video membership platform with branded native apps (iOS, Android, Roku, Apple TV), live streaming, and community features. Netflix-style content delivery.
- **API Documentation:** https://api-docs.uscreen.cloud/ (Publisher API v2)
- **SDKs/Libraries:** No official SDKs
- **Developer Guide:** https://help.uscreen.tv/en/articles/4576093-uscreen-api
- **Standards:** REST/JSON, API Key authentication
- **Authentication:** API Key in request headers

### Passes
- **Description:** Creator accelerator platform supporting subscriptions, paid DMs, livestreams, merchandise, one-on-one calls, and content protection. 10% platform fee with 90% revenue retention.
- **API Documentation:** No public developer documentation available
- **SDKs/Libraries:** N/A
- **Developer Guide:** N/A
- **Standards:** Proprietary — no documented API standards
- **Authentication:** N/A

### Buy Me a Coffee
- **Description:** Fan support platform with one-time tips, memberships, and livestream integration. 5% flat fee with no paid plans or premium tiers.
- **API Documentation:** https://developers.buymeacoffee.com/
- **SDKs/Libraries:** No official SDKs; community Laravel, Lambda implementations on GitHub
- **Developer Guide:** https://studio.buymeacoffee.com/webhooks/docs
- **Standards:** REST/JSON, Webhooks (HTTP POST with 2xx acknowledgement)
- **Authentication:** API tokens; webhook verification

### Teachable
- **Description:** Online course platform with video lessons, quizzes, coaching, affiliate marketing, and multi-language subtitle support across 130+ currencies.
- **API Documentation:** https://docs.teachable.com/
- **SDKs/Libraries:** No official SDKs
- **Developer Guide:** https://docs.teachable.com/docs/overview
- **Standards:** REST/JSON, Webhooks
- **Authentication:** API access on Pro and Business plans only

### Stripe Connect (Payment Infrastructure)
- **Description:** Payment platform powering the majority of creator monetisation platforms. Handles marketplace splits, creator payouts, subscription billing, and tax compliance.
- **API Documentation:** https://docs.stripe.com/connect
- **SDKs/Libraries:** Node.js (v21.0.1), Python (v15.0.0), Ruby, PHP, Java, .NET, Go — all officially maintained
- **Developer Guide:** https://docs.stripe.com/connect/end-to-end-marketplace
- **Standards:** REST/JSON, OpenAPI, Accounts v2 API, 135+ currencies, 40+ payment methods
- **Authentication:** API Keys (publishable + secret), OAuth 2.0 for Connect account onboarding

## Notes

- **API maturity varies significantly**: Stripe provides the most comprehensive developer experience. Patreon, Kajabi, Gumroad, Fourthwall, and Teachable offer documented REST APIs. Ko-fi and Buy Me a Coffee are webhook-only. Substack and Passes have no meaningful public API.

- **Standard Webhooks adoption is emerging**: The Standard Webhooks specification provides a path toward consistent event notification delivery, but none of the analysed platforms have formally adopted it yet. This represents an opportunity for differentiation.

- **MCP integration is a greenfield opportunity**: No existing creator monetisation platform exposes an MCP server. Building MCP-compatible tools from day one would position an AI-native platform ahead of incumbents for LLM-powered workflows.

- **Privacy regulation is fragmenting**: Beyond GDPR and CCPA, over 20 US states have enacted privacy laws as of 2026. A creator platform must implement jurisdiction-aware consent management rather than a one-size-fits-all approach.

- **PCI DSS v4.1 shifts toward continuous compliance**: Platforms relying on Stripe Connect can largely delegate PCI compliance to Stripe, but must still maintain SAQ-A compliance for their own checkout flows and ensure no cardholder data touches their servers.
