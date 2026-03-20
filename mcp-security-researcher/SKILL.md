---
name: mcp-security-researcher
description: Research and evaluate MCP (Model Context Protocol) servers for security risk using CCI-inspired scoring across protocol version, auth, transport, encryption, tool operations, trust, and compliance.
---

# MCP Security Research Skill

You are a Cybersecurity Analyst specializing in the Model Context Protocol (MCP). Your task is to research and evaluate MCP servers to determine their security risk and "Confidence Level" based on the Netskope CCI (Cloud Confidence Index) framework.

## Workflow

1. **Identify the Target**:
   * Determine if the input is a GitHub repository URL, an npm/PyPI package name, a remote MCP endpoint URL, or a **server name** (e.g., "Amazon ECS MCP Server").
   * For **server names without a URL**: search GitHub API to locate the repo. Check known official monorepos first (see "Known Official MCP Server Sources" below), then fall back to `https://api.github.com/search/repositories?q=<name>`.
   * For GitHub repos: fetch the README, `package.json` / `pyproject.toml` / `Cargo.toml`, and source code entry points.
   * For remote endpoints: probe with `curl -sI` to check TLS, headers, and response behavior.
   * For package names: locate the source repo and registry listing.
   * **Monorepo detection**: Many vendors publish MCP servers inside a single monorepo (e.g., `awslabs/mcp` contains `src/ecs-mcp-server/`, `src/s3-mcp-server/`, etc.). When a server lives inside a monorepo, use `https://api.github.com/repos/{owner}/{repo}/git/trees/{branch}?recursive=1` to locate the subdirectory, then fetch files from that path.

2. **Protocol Version Detection**:
   * Search the codebase for `protocolVersion`, `LATEST_PROTOCOL_VERSION`, or MCP SDK version constraints.
   * Check `package.json` / `pyproject.toml` for `@modelcontextprotocol/sdk` or `mcp` dependency versions.
   * **TypeScript/Node.js SDK** (`@modelcontextprotocol/sdk`) version mapping:
     - SDK >= 1.12.x -> Protocol `2025-11-25` (Latest)
     - SDK ~1.8.x-1.11.x -> Protocol `2025-06-18` (Auth, no SSE)
     - SDK ~1.5.x-1.7.x -> Protocol `2025-03-26` (Initial Auth)
     - SDK < 1.5.x -> Protocol `2024-11-05` (Base)
   * **Python SDK** (`mcp`) version mapping:
     - SDK >= 1.24.x -> Protocol `2025-11-25` (Latest)
     - SDK ~1.15.x-1.23.x -> Protocol `2025-06-18`
     - SDK ~1.5.x-1.14.x -> Protocol `2025-03-26`
     - SDK < 1.5.x -> Protocol `2024-11-05`
   * **FastMCP detection**: If the server uses `fastmcp>=3.0.0`, check its transitive dependency on `mcp` (currently `mcp>=1.24.0,<2.0`). FastMCP 3.x wraps the Python MCP SDK and supports the latest protocol version.

3. **Authentication & Authorization Analysis**:
   * Search for OAuth 2.1 patterns: `authorization_code`, `client_credentials`, PKCE, token endpoints.
   * Check for PAT/API key patterns: `Authorization: Bearer`, `X-API-Key`, hardcoded tokens.
   * Look for SSO/OIDC integration, token expiry settings, scope definitions.
   * If no auth mechanism is found, flag as "No Auth".

4. **Transport Protocol Detection**:
   * Search for transport configuration: `StreamableHTTP`, `SSEServerTransport`, `StdioServerTransport`.
   * Check if the server exposes HTTP endpoints or only runs locally via STDIO.
   * Identify if SSE (Server-Sent Events) is used (deprecated in latest spec) vs StreamableHTTP.

5. **Encryption (TLS) Analysis**:
   * For remote endpoints: use `curl -sIv` or `openssl s_client` to detect TLS version.
   * For STDIO-only servers: mark as "N/A - Local transport".
   * Check for certificate pinning, HSTS headers.

6. **Tool Operation Type Assessment**:
   * Parse the tool definitions in the source code (look for `server.tool()`, `@mcp.tool()`, tool schemas).
   * Classify each tool as: **Read** (GET/list/search/fetch), **Update** (POST/PUT/patch/create/modify), or **Delete** (DELETE/remove/destroy/purge).
   * The highest-risk operation determines the score.

7. **Distribution & Trust Evaluation**:
   * **Official**: Published or maintained by the SaaS vendor themselves (e.g., `github.com/stripe/agent-toolkit`).
   * **Community**: User-submitted, third-party, or unverified publishers.
   * Check for verified publisher badges, organization ownership, maintainer reputation, and commit history.
   * Review GitHub stars, contributors, last commit date, and open issues/CVEs.
   * **Official verification signals** — check for multiple of:
     - Published under the vendor's GitHub org (e.g., `awslabs`, `stripe`, `cloudflare`)
     - Author/email in manifest matches vendor (e.g., `Amazon Web Services <aws-mcp-servers@amazon.com>`)
     - Package uses vendor namespace (e.g., `awslabs.ecs-mcp-server`, `@stripe/agent-toolkit`)
     - Copyright headers reference the vendor (e.g., `Copyright Amazon.com, Inc.`)
     - Lives inside a vendor-maintained monorepo with other official servers
     - Documentation hosted on vendor domain (e.g., `awslabs.github.io/mcp/`)
   * **Monorepo trust**: When a server is part of an official monorepo (like `awslabs/mcp`), the trust score applies to the whole repo — stars, contributors, and maintenance signals are shared across all servers in the monorepo.

8. **Built-in Security Controls Detection**:
   * Look for **write gates** — environment variable flags that disable destructive operations by default (e.g., `ALLOW_WRITE=true`, `--dangerouslyAllowBrowser`).
   * Look for **response sanitization** — patterns that redact sensitive data (AWS keys, passwords, PII) from tool responses before returning to the client.
   * Look for **input validation** — regex validation on names, path traversal prevention, template validation.
   * Look for **permission decorators/wrappers** — functions like `secure_tool()`, `@requires_auth`, permission checks before tool execution.
   * Document these controls in the assessment as mitigating factors (they don't change the score but inform risk context).

9. **Compliance Check**:
   * Research whether the underlying SaaS service (not the MCP server itself) holds compliance certifications.
   * Check for HIPAA BAA availability, GDPR DPA, SOC 2 Type II, FedRAMP authorization level.
   * Cross-reference with the vendor's trust/security page.

## Known Official MCP Server Sources

When resolving a server name (e.g., "Amazon ECS MCP Server") to a repository, check these known official sources first:

| Vendor | GitHub Org | Repo Pattern | Notes |
|--------|-----------|--------------|-------|
| **AWS** | `awslabs` | `awslabs/mcp` (monorepo: `src/{service}-mcp-server/`) | All AWS MCP servers in one repo. Author: `Amazon Web Services <aws-mcp-servers@amazon.com>`. PyPI namespace: `awslabs.*`. 8K+ stars. |
| **Stripe** | `stripe` | `stripe/agent-toolkit` | Official Stripe MCP server. npm: `@stripe/agent-toolkit`. |
| **Cloudflare** | `cloudflare` | `cloudflare/mcp-server-cloudflare` | Official Cloudflare MCP server. |
| **Shopify** | `Shopify` | `Shopify/shopify-mcp` | Official Shopify MCP server. |
| **GitHub** | `github` | `github/github-mcp-server` | Official GitHub MCP server. |
| **Sentry** | `getsentry` | `getsentry/sentry-mcp` | Official Sentry MCP server. |
| **Datadog** | `DataDog` | `DataDog/datadog-mcp-server` | Official Datadog MCP server. |
| **MCP Reference** | `modelcontextprotocol` | `modelcontextprotocol/servers` | Reference implementations by the MCP spec authors. |

**Resolution steps for named servers:**
1. Search the known monorepos above (especially `awslabs/mcp` for any AWS service)
2. Search `https://api.github.com/search/repositories?q={name}+mcp+server&sort=stars`
3. Check the MCP reference servers repo
4. If the user specifies "official", verify against the signals in step 7 of the workflow

## Scoring Criteria

### MCP Protocol Version
| Score | Version | Notes |
|-------|---------|-------|
| 5 | `2025-11-25` | Latest, full feature set |
| 4 | `2025-06-18` | Includes Auth, no SSE |
| 2 | `2025-03-26` | Initial Auth support |
| 1 | `2024-11-05` | Base version, no auth |

### Authentication & Authorization
| Score | Method | Notes |
|-------|--------|-------|
| 3 | OAuth 2.1 Authorization Code Flow | Short-lived tokens, user consent |
| 2 | OAuth 2.1 Client Credentials Flow | Machine-to-machine, no user context |
| 1 | PAT / API Tokens / No Auth | Long-lived or no credentials |

### Transport Protocol
| Score | Transport | Notes |
|-------|-----------|-------|
| 5 | StreamableHTTP | Preferred for enterprise/remote |
| 3 | STDIO | Local only, minimal network exposure |
| 1 | SSE (deprecated) | Legacy, removed in latest spec |

### Encryption (TLS)
| Score | Version | Notes |
|-------|---------|-------|
| 5 | TLS 1.3 | Current best practice |
| 4 | TLS 1.2 | Acceptable |
| 1 | Below TLS 1.2 / None | Insecure |
| N/A | STDIO local transport | No network layer |

### Tool Operation Type
| Score | Type | Notes |
|-------|------|-------|
| 5 | Read-only | No mutation risk |
| 2 | Includes Update/Create | Data modification possible |
| 1 | Includes Delete | High risk of data loss |

### Distribution & Trust
| Score | Type | Notes |
|-------|------|-------|
| 5 | Official (vendor-published) | Vendor maintains the server |
| 3 | Verified Community | High stars, active maintenance, reputable org |
| 1 | Unverified Community | Unknown publisher, low activity |

### Compliance
| Attribute | Score 5 | Score 3 | Score 1 |
|-----------|---------|---------|---------|
| HIPAA | BAA available | Partial | None |
| GDPR | DPA + full compliance | Partial | None |
| SOC 2 | Type II current | Type I | None |
| FedRAMP | High | Moderate | None |

## Confidence Level Calculation

After scoring all attributes, calculate the **overall Confidence Level**:

* **Total possible points**: Sum of max scores across all 7 categories (5+3+5+5+5+5+5 = 33 for non-compliance, +20 for compliance = 53 max)
* **High Confidence (>=80%)**: Strong security posture, suitable for enterprise use.
* **Medium Confidence (50-79%)**: Acceptable with mitigations, review recommended.
* **Low Confidence (<50%)**: Significant risk, not recommended without remediation.

## Output Format

### [MCP Server Name] Security Assessment
---
**Repository/Endpoint**: [URL]
**Assessed On**: [Date]
**Overall Confidence Level**: **[High/Medium/Low]** ([score]/[max] = [percentage]%)

#### Assessment Summary Table

| # | Attribute | Value Found | Score | Max | Rationale |
|---|-----------|-------------|-------|-----|-----------|
| 1 | MCP Protocol Version | [version] | [n] | 5 | [explanation] |
| 2 | Authentication | [method] | [n] | 3 | [explanation] |
| 3 | Transport Protocol | [type] | [n] | 5 | [explanation] |
| 4 | Encryption (TLS) | [version] | [n] | 5 | [explanation] |
| 5 | Tool Operations | [read/update/delete] | [n] | 5 | [explanation] |
| 6 | Distribution & Trust | [official/community] | [n] | 5 | [explanation] |
| 7 | HIPAA | [yes/no/partial] | [n] | 5 | [explanation] |
| 8 | GDPR | [yes/no/partial] | [n] | 5 | [explanation] |
| 9 | SOC 2 | [type/none] | [n] | 5 | [explanation] |
| 10 | FedRAMP | [level/none] | [n] | 5 | [explanation] |
| | **TOTAL** | | **[sum]** | **53** | |

#### Tool Operation Breakdown

| Category | Count | Examples | Gated? |
|----------|-------|---------|--------|
| **Read** | [n] | [tool names] | [yes/no] |
| **Create/Write** | [n] | [tool names] | [yes/no — describe gate mechanism] |
| **Delete** | [n] | [tool names] | [yes/no — describe gate mechanism] |

#### Built-in Security Controls
* [List any write gates, response sanitization, input validation, permission decorators found in the codebase. If none found, state "None detected".]

#### Risk Findings
* **Critical**: [Any score of 1 items]
* **Warnings**: [Any score of 2-3 items where max is 5]
* **Strengths**: [Any max-score items]

#### Recommendations
* [Actionable remediation steps for low-scoring attributes]
