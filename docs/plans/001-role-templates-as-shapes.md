---
summary: Network role templates as shapes — per-organization roles based on them, adjustments that survive upgrades, materialized into ordinary @_linked/access grants
packages: [access-templates, access, org, core]
---

# 001 — Role templates as shapes

## 1. Why this package exists

`@_linked/access` evaluates grants: an assignee may perform actions on a target while conditions hold. Grants
are shapes (`cnacl:AccessGrant`), stored as triples, and the evaluator is complete for what an app needs
(assignee = actor or any id the host puts in the context, selectors down to shape/instance/property,
fail-closed resolver conditions, deny wins, attenuation).

**Templates are not.** They are a hard-coded TypeScript constant of four Create Now ids
(`viewer | reviewer | mapper | admin`), each one flat action list, producing ONE grant per scope and assignee.
An app cannot register its own, a template cannot span several targets, and nothing records that an
organization's role was based on one.

Serve — and any network of organizations built on LINKED — needs the opposite balance: **consistency across
the network, with flexibility for each organization**. Every organization should start from the same
standard roles, may adjust them and add its own, and both the organization and the network should be able to
see where it differs.

This package makes templates and roles **data**, like everything else in LINKED, and materializes them into
ordinary grants. The evaluator never reads a template (as today): it only ever sees grants.

## 2. The model

```
RoleTemplate  (network, versioned)          e.g. "Manager" v3, published by Serve
  └─ TemplateRule*                          what · which actions · permit|prohibit
        ▲ basedOn (template + version)
OrganizationRole  (one organization's)      an @_linked/org Role belonging to one organization
  └─ RoleAdjustment*                        + or − a rule, pinned; survives template upgrades
        │ materialize
        ▼
AccessGrant*  (@_linked/access)             assignee = the OrganizationRole
                                             target  = the rule's selector
                                             condition = boundary(within: the organization)
```

### Shapes

| Shape | Class | Key properties |
|---|---|---|
| `RoleTemplate` | `acct:RoleTemplate` | `key` (stable, e.g. `serve.manager`), `version` (integer), `label`, `description`, `publishedBy` (the app/package), `level` (`owner \| manager \| moderator \| member \| post`), `rules` (`contains`) |
| `TemplateRule` | `acct:Rule` (`dependent`) | `effect` (`permit \| prohibit`), `actions` (from the `@_linked/access` action vocabulary), `target` (a selector: capability, shape class, or property), `conditions` (extra, e.g. a `vc` requirement) |
| `OrganizationRole` | extends `@_linked/org` `Role` | `roleIn` (the organization), `basedOn` (a `RoleTemplate`) + `basedOnVersion`, `label` (may rename), `level`, `createdBy`, `adjustments` (`contains`) |
| `RoleAdjustment` | `acct:Adjustment` (`dependent`) | a `TemplateRule`-shaped rule plus `op` (`add \| remove`), `pinnedBy`, `pinnedAt`, `reason` |

A custom role is an `OrganizationRole` with no `basedOn` (or based on a template and heavily adjusted) — one
model, not two. Memberships point at an `OrganizationRole` with `org:role`, exactly as they point at any
`org:Role` (`@_linked/org` `Membership`).

### Why the assignee is the organization's role

`@_linked/access` collects grants whose assignee is the actor or any id in `context.memberships`, which the
HOST builds. The host adds the `OrganizationRole`s a person holds (through their memberships) to that list.
A grant assigned to an `OrganizationRole` therefore reaches everyone holding it — **one grant per rule per
organization, never one per person**. It is safe precisely because the role belongs to one organization; a
grant on a SHARED role node would speak for everyone everywhere, which is why hosts must never offer shared
roles as assignees.

The `boundary` condition (`within: <organization>`) is answered by the host's `ContainmentOracle` — "is this
instance inside that organization?" — which is data, not naming (the evaluator deliberately refuses to infer
it from the selector hierarchy).

### Do NOT use the `role` condition for per-organization rules

`AccessCondition { kind: 'role' }` is satisfied if ANY of the actor's memberships has that role, anywhere. A
manager elsewhere would pass "role ≥ admin" here. Per-organization authority comes from the assignee plus the
boundary, never from `role`.

## 3. Operations

| Operation | What it does |
|---|---|
| `publishTemplate(template)` | An app registers or versions a template (data). A new version never mutates organizations by itself. |
| `instantiateRoles(organization, templates)` | On organization creation: one `OrganizationRole` per template, and its grants. |
| `adjustRole(role, adjustment, actor)` | Pin an add/remove. Checked against guardrails (§4). Re-materializes that role's grants. |
| `createRole(organization, {label, level, basedOn?, rules}, actor)` | A new role, from a template or from nothing. |
| `diffRole(role)` | Template rules vs effective rules: what this organization added and removed. |
| `previewUpgrade(template, toVersion)` | For every organization based on it: what would change; pinned adjustments kept and listed. |
| `applyUpgrade(template, toVersion, organization?)` | Re-materialize the unadjusted part. Never overwrites a pin. |
| `networkReport(template)` | How many organizations use it; how many unchanged; the most common adjustments — the signal that the standard itself should change. |
| `materialize(role)` | Rules → grants, deterministic ids (re-materializing replaces, never duplicates); removed rules revoke (an event, kept as history). |

## 4. Guardrails (checked on write, never trusted from the client)

1. **Attenuation** — a person cannot create or adjust a role to carry more than they themselves hold.
2. **Only an owner** may give a role `grants.manage`, or adjust the owner role.
3. **The owner role cannot be reduced below managing roles** — an organization can never lock itself out.
4. **Levels bound assignment** — the role's `level` decides who may hand it out (the host's ladder; e.g. a
   manager can assign a moderator-level role, never a manager-level one).
5. **Deleting a role** revokes its grants; memberships holding it fall back to nothing (never up).
6. **Unknown actions or selectors are refused**, never stored — `assertActionName` at the boundary.

The package enforces 1, 2, 3, 5, 6. Rule 4 needs the host's role ladder, supplied as a function.

## 5. What the host (the app) supplies

- Its templates (`publishTemplate`).
- The context builder: memberships, the `OrganizationRole`s they hold, posts if it models them.
- The `ContainmentOracle` for `boundary`, and any `vc` / `chain` resolvers it wants.
- The role ladder for guardrail 4.

## 6. Non-goals

- The evaluator, the grant store and the contract stay in `@_linked/access`, unchanged.
- Identity and membership stay in `@_linked/org`.
- No UI in this package; `diffRole` / `previewUpgrade` / `networkReport` return data for an app's screens.

## 7. Dependencies and compatibility

`@_linked/core` (^2.22.8), `@_linked/org` (^1.2.1), `@_linked/access` (^0.1.1). Create Now's four hard-coded
templates can later be published through this package, giving one template system — proposed, not assumed.

## 8. Open questions (for René)

1. Namespace for the new classes (`acct:` here is a placeholder).
2. Should `OrganizationRole` live here, or be proposed to `@_linked/org` (it is an `org:Role` with `roleIn`)?
3. Should `@_linked/access` eventually read `basedOn` to render "Manager (standard, 1 adjustment)" in its own
   effective-access screens, or should that stay in each app?

## 9. Build order

1. Shapes + ontology + `publishTemplate` / `instantiateRoles` / `materialize`, with an in-memory policy
   repository in tests.
2. Adjust / create / guardrails.
3. Diff, preview, apply upgrade, network report.
4. First consumer: Serve (its WP57).
