# ADR: Schema Prefix Support for `pluginDivisionMode: schema`

## Context

**Problem**: Strategic customer on RHDH 1.8.5 is blocked from deploying due to PostgreSQL schema conflict with existing pgvector installation.

**Support Case**: [RHDHSUPP-404](https://redhat.atlassian.net/browse/RHDHSUPP-404) - RHDH 1.8.5 Upgrade Failure Due to Conflict with Existing PostgreSQL "extensions" Schema Used by pgvector

**Support Exception**: Extended support approved for RHDH 1.8.5 (EOL)

**Business Impact**:
- Strategic customer blocked from production deployment
- Customer renewal at risk (end of year)
- DBA policies prohibit workarounds (cannot rename existing `extensions` schema used by pgvector)
- Requested urgent prioritization as this blocks their upgrade path

**Technical Root Cause**:

Backstage's database manager ([@backstage/backend-defaults](https://github.com/backstage/backstage/tree/master/packages/backend-defaults)) has an inconsistency in how it handles database vs schema naming when using `pluginDivisionMode: schema`:

- **Database names**: Support configurable prefix via [`backend.database.prefix`](https://backstage.io/docs/tutorials/configuring-plugin-databases#custom-database-name-prefix) (defaults to `backstage_plugin_`)
  - Implementation: [DatabaseManager.ts:257](https://github.com/backstage/backstage/blob/master/packages/backend-defaults/src/entrypoints/database/DatabaseManager.ts#L257)
  
- **Schema names**: Hardcoded to raw plugin IDs with **no prefix support**
  - Implementation: [postgres.ts:641](https://github.com/backstage/backstage/blob/master/packages/backend-defaults/src/entrypoints/database/connectors/postgres.ts#L641) - `searchPath: [pluginId]`
  - Schema creation: [postgres.ts:703](https://github.com/backstage/backstage/blob/master/packages/backend-defaults/src/entrypoints/database/connectors/postgres.ts#L703)

When using `pluginDivisionMode: schema`, RHDH creates PostgreSQL schemas named after plugin IDs (e.g., `catalog`, `auth`, `extensions`). The `extensions` schema name conflicts with the customer's existing `extensions` schema used for PostgreSQL extensions like [pgvector](https://github.com/pgvector/pgvector).

**Error observed**:
```
Failed to instantiate service 'core.httpRouter' for 'extensions'
permission denied for schema extensions
```

The RHDH application role lacks permissions to create objects in the existing `extensions` schema owned by the database-owner account.

**Customer Configuration**:
```yaml
backend:
  database:
    pluginDivisionMode: schema  # Required for their architecture
    role: ${DB_CONNECTION_ROLE}
    connection:
      # AWS RDS PostgreSQL with pgvector
```

**Why Workarounds Are Not Acceptable**:

1. **Renaming pgvector schema**: Rejected by DBA team - violates database governance policies
2. **Removing `pluginDivisionMode: schema`**: Customer requirement for single-database deployment architecture
3. **Granting schema ownership to app role**: Violates RBAC security policies

**Upstream Status**:

- Issue filed: [backstage/backstage#35296](https://github.com/backstage/backstage/issues/35296) - "Support Schema Name Prefix Configuration for `pluginDivisionMode: schema`"
- Maintainer feedback received from @awanlin requesting:
  1. Migration story for existing deployments
  2. Validation for PostgreSQL 63-character limit
- Both questions answered in [our response](https://github.com/backstage/backstage/issues/35296#issuecomment-<response-id>)
- No PR submitted yet (awaiting this ADR approval before upstream implementation)
- Similar requests previously closed as "not planned": [#15914](https://github.com/backstage/backstage/issues/15914), [#13415](https://github.com/backstage/backstage/issues/13415), [#17838](https://github.com/backstage/backstage/issues/17838)

## Decision

**Implement schema prefix support as a downstream patch in RHDH 1.8.x** to immediately unblock the customer, while **continuing upstream contribution in parallel**.

### Implementation Approach

**Use Yarn 4's native patching feature** to patch `@backstage/backend-defaults@0.17.7` in RHDH release-1.8.6:

1. **Add `schemaPrefix` configuration option** to `backend.database` config (similar to existing `prefix` option)
2. **Patch two files** in `@backstage/backend-defaults`:
   - `config.d.ts` - Add TypeScript type definition for `schemaPrefix`
   - `src/entrypoints/database/connectors/postgres.ts` - Apply prefix to schema names with validation
3. **Generate single patch file**: `.yarn/patches/@backstage+backend-defaults+npm+0.17.7+<hash>.patch`
4. **Document in RHDH docs**: Create `docs/database-schema-prefix.md` for customer guidance

**Patch Content** (applied to upstream Backstage code in RHDH's node_modules):

```typescript
// config.d.ts - Add after line 682
/**
 * Schema name prefix for pluginDivisionMode: schema
 * Only applies when pluginDivisionMode is set to 'schema'
 * @default ''
 */
schemaPrefix?: string;
```

```typescript
// postgres.ts - In computePgPluginConfig() function
const schemaPrefix = config.getOptionalString('schemaPrefix') || '';

// Validate PostgreSQL 63-character limit
if (pluginDivisionMode === 'schema') {
  const schemaName = `${schemaPrefix}${pluginId}`;
  if (schemaName.length > 63) {
    throw new Error(
      `PostgreSQL schema name "${schemaName}" exceeds 63-character limit. ` +
      `Consider using shorter schemaPrefix (current: "${schemaPrefix}") or plugin ID.`
    );
  }
}

// Apply prefix to searchPath (line 641)
databaseClientOverrides = mergeDatabaseConfig({}, databaseClientOverrides, {
  searchPath: [`${schemaPrefix}${pluginId}`],  // Changed from: [pluginId]
});
```

**Customer Configuration** (after patch applied):

```yaml
backend:
  database:
    pluginDivisionMode: schema
    schemaPrefix: 'rhdh_'  # NEW - resolves conflict!
    connection:
      # ... existing connection settings
```

**Result**: Schemas created as `rhdh_catalog`, `rhdh_auth`, `rhdh_extensions` - no conflict with existing `extensions` schema.

### Why Yarn Patching?

RHDH already uses Yarn 4 native patching (documented in [docs/patch-package.md](https://github.com/redhat-developer/rhdh/blob/main/docs/patch-package.md)), with existing patches in `.yarn/patches/`. This approach:

- ✅ **Minimal code changes** - One patch file, no new TypeScript files
- ✅ **Reversible** - Delete patch when upstream merges
- ✅ **Established workflow** - Follows RHDH's existing patching process
- ✅ **Automatic application** - Yarn applies on `yarn install`
- ✅ **Version controlled** - Patch file committed to repository

### Target Release

**RHDH 1.8.6** - Create release from `release-1.8` branch (customer has support exception for 1.8.x)

## Alternatives Considered

### Alternative 1: Wait for Upstream PR to Merge

- **Approach**: Submit PR to Backstage, wait for review/merge/release, then pull into RHDH
- **Timeline**: Minimum 4-8 weeks (PR review → upstream release → RHDH incorporation)
- **Rejected because**: 
  - Customer blocked NOW - renewal at risk end of year
  - Upstream has previously closed similar requests as "not planned" (#15914, #13415, #17838)
  - No guarantee of acceptance timeline
  - Even after upstream merge, customer still needs RHDH 1.8.x backport (EOL branch)

### Alternative 2: Custom Database Service Factory in RHDH

- **Approach**: Create new TypeScript files in RHDH (`packages/backend/src/services/database.ts`, `packages/backend/src/services/postgres-connector.ts`) that override upstream implementation
- **Rejected because**:
  - More code to maintain (multiple new files vs one patch)
  - Harder to remove when upstream merges (must revert imports, delete files)
  - Creates larger divergence from upstream
  - Previous analysis showed this as more complex than patching

### Alternative 3: Recommend Customer Remove `pluginDivisionMode: schema`

- **Approach**: Have customer use default `pluginDivisionMode: database` (separate database per plugin)
- **Rejected because**:
  - Customer architecture requires single-database deployment
  - Changing their architecture is not acceptable
  - This is a valid use case that Backstage should support

### Alternative 4: Fork @backstage/backend-defaults

- **Approach**: Maintain RHDH-specific fork of backend-defaults package
- **Rejected because**:
  - Significant maintenance burden (track upstream changes, merge conflicts)
  - Creates major divergence from upstream
  - Harder to justify to community
  - Goes against RHDH's goal of staying aligned with upstream Backstage

### Alternative 5: Database-Level Workaround (Grant Schema Permissions)

- **Approach**: Grant RHDH application role CREATE permissions on existing `extensions` schema
- **Rejected because**:
  - Violates customer's RBAC and security policies
  - DBA team will not grant app ownership of infrastructure schema
  - Does not solve the root problem (hard-coded schema names)

## Consequences

### Positive

- ✅ **Customer immediately unblocked** - Can deploy RHDH 1.8.5 to production
- ✅ **Minimal code changes** - Single patch file, no custom TypeScript
- ✅ **Easy cleanup** - Delete patch when upstream merges
- ✅ **Proven workflow** - Uses established RHDH patching process
- ✅ **Backward compatible** - Default empty prefix maintains current behavior
- ✅ **No breaking changes** - Existing deployments unaffected
- ✅ **Solves whole class of conflicts** - Works for any schema name collision
- ✅ **Aligns with upstream pattern** - Mirrors existing `database.prefix` design

### Negative

- ❌ **Downstream maintenance burden** - Must maintain patch until upstream merges
- ❌ **Upgrade risk** - Patch may conflict when upgrading @backstage/backend-defaults
- ❌ **Team tracking overhead** - Must monitor upstream for merge, then remove patch
- ❌ **Temporary divergence** - RHDH 1.8.x has feature upstream doesn't (until merged)
- ❌ **Documentation duplication** - Must document in both RHDH and (eventually) upstream

### Neutral

- ⚖️ **Applies only to RHDH 1.8.x** - Will be obsolete when customer upgrades to future RHDH version with upstream fix
- ⚖️ **Patch will be redundant** - Feature becomes unnecessary once upstream merges and RHDH pulls it in
- ⚖️ **No impact on other RHDH versions** - Patch isolated to 1.8.x branch

## Parallel Activities

This ADR approves a **two-track approach** to maximize value:

### Track 1: Downstream Patch (This ADR)

**Goal**: Immediate customer relief

**Timeline**: 5 days to customer deployment

**Steps**:
1. Generate Yarn patch for @backstage/backend-defaults@0.17.7
2. Test locally with PostgreSQL + pgvector conflict scenario
3. Create customer documentation (`docs/database-schema-prefix.md`)
4. Build RHDH 1.8.6 release
5. Deploy to customer staging → production

**Deliverable**: RHDH 1.8.6 with schema prefix support

### Track 2: Upstream Contribution (Parallel)

**Goal**: Long-term solution for Backstage community

**Timeline**: Submit PR after ADR approval, upstream timeline TBD

**Steps**:
1. Implement same changes in upstream Backstage (main branch)
2. Add comprehensive tests (unit + integration)
3. Update upstream documentation
4. Submit PR referencing issue #35296
5. Address maintainer feedback
6. Wait for merge and release

**Deliverable**: Upstream Backstage with schema prefix support

### Convergence Plan

**When upstream merges**:
1. RHDH pulls new upstream version
2. Remove downstream patch from RHDH 1.8.x (for continuity)
3. Future RHDH releases use upstream implementation
4. Customer upgrades to newer RHDH version with native support

## Migration and Compatibility

**For new deployments**:
- Set `schemaPrefix` in initial configuration
- Schemas created with prefix from start

**For existing RHDH deployments** (without prefix, wanting to add one):
- **Option 1** (Recommended): Fresh database with prefix configured
- **Option 2**: In-place - existing schemas stay unprefixed, new plugins get prefix (document ownership)
- **Option 3**: Manual schema rename + data migration (DBA effort)

**Validation**:
- PostgreSQL 63-character limit enforced with clear error message
- Startup failure if `schemaPrefix + pluginId > 63 characters`

## Success Criteria

- [ ] ADR approved by Engineering, Product Management, and Support teams
- [ ] Patch file generated and tested
- [ ] RHDH 1.8.6 builds successfully with patch
- [ ] Local testing passes (PostgreSQL + pgvector conflict scenario)
- [ ] Customer successfully deploys to staging
- [ ] Customer confirms production deployment success
- [ ] No schema conflicts observed
- [ ] Backward compatibility verified (no prefix = current behavior)
- [ ] Documentation complete and clear
- [ ] Upstream PR submitted to backstage/backstage#35296

## Timeline

- **Day 1**: ADR approval → Generate patch → Create documentation
- **Day 2**: Local testing with PostgreSQL + pgvector
- **Day 3**: Build RHDH 1.8.6 → Customer staging deployment
- **Day 4**: Customer validation
- **Day 5**: Customer production deployment
- **Parallel**: Submit upstream PR (no dependency on customer deployment)

## References

- **Support Case**: [RHDHSUPP-404](https://redhat.atlassian.net/browse/RHDHSUPP-404)
- **Support Exception**: Approved for RHDH 1.8.5
- **Upstream Issue**: [backstage/backstage#35296](https://github.com/backstage/backstage/issues/35296)
- **Similar Closed Issues**: [#15914](https://github.com/backstage/backstage/issues/15914), [#13415](https://github.com/backstage/backstage/issues/13415), [#17838](https://github.com/backstage/backstage/issues/17838)
- **Upstream Response**: [Issue #35296 Comment](https://github.com/backstage/backstage/issues/35296#issuecomment-<response-id>)
- **RHDH Patching Docs**: [docs/patch-package.md](https://github.com/redhat-developer/rhdh/blob/main/docs/patch-package.md)
- **Database Config Docs**: [Configuring Plugin Databases](https://backstage.io/docs/tutorials/configuring-plugin-databases)
- **PostgreSQL Schema Limits**: [PostgreSQL Identifier Length](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS)
- **Implementation Plan**: `RHDHSUPP-404-rhdh-1-8-5-upgrade-failure-due-to-conflict-with-existing-postgre-sql-extensions-schema-used-by-pgvector/implementation-plan.md`
