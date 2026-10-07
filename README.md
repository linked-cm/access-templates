# @linked.cm/access-templates

Network role templates for [`@linked.cm/access`](https://github.com/linked-cm/access).

An app publishes standard role templates (Owner, Manager, Moderator, …) as **shapes**. Each organization gets its own
roles based on them, can adjust them or add its own, and those adjustments survive template upgrades. Roles are
materialized into ordinary `@linked.cm/access` grants. The evaluator never reads a template, only grants.

The goal is consistency across a network of organizations, with room for each one to differ, and with the
differences visible to the organization and to the network.

**Status: design only.** Nothing is implemented yet. Start with
[`docs/plans/001-role-templates-as-shapes.md`](docs/plans/001-role-templates-as-shapes.md).

Staged in the **linked-cm** org (npm scope `@linked.cm`) pending René's review. It is not published: there is
deliberately no `publish.yml` yet, so landing `main` cannot release it. The first publish needs René.

## License

MIT © Semantu
