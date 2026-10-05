# Agent Users

Agent Users gives external software—AI agents, MCP clients, scheduled jobs, and similar tools—a dedicated WordPress identity. The goal is to make its work attributable and independently revocable instead of sharing a human account.

This experiment implements the identity model proposed in [WordPress/ai#923](https://github.com/WordPress/ai/issues/923). Audit trails and richer provenance can build on that identity separately.

## Design

An agent is a regular WordPress user marked with `wpai_agent` user meta. Reusing `WP_User` preserves the behavior the ecosystem already expects: roles and capabilities, content authorship, revisions, comments, deletion with content reassignment, and user-based logs.

Every agent is the child of a human **parent** account it acts on behalf of, stored in `wpai_agent_parent` user meta. The agent keeps its own identity, so its work is attributed to the agent, and the link lets that work be shown as "Agent on behalf of Parent". The provisioner is recorded separately in `wpai_agent_created_by`, because an administrator may create an agent for another user.

The link is user meta rather than a new `wp_users` column: a plugin should not alter core's schema, and user meta is network-wide on multisite, exactly like the agent marker. A column only makes sense if the model is proposed for core.

The marker changes the account's security contract:

- **Authentication is non-interactive.** Password login and password resets are blocked. Credentials are issued and revoked through core's Application Password flows under its normal permission checks. Other authentication mechanisms may be used if they resolve the request to the agent, because the identity restrictions are applied to the resulting WordPress user rather than to one credential format.
- **The parent is the ceiling.** Every capability check for an agent also checks its parent for the same capability with the same arguments, such as the post being edited, after every other mapping has run. An agent's authority is therefore both its own role and its parent's *current* permissions. Demoting the parent narrows their agents immediately, and object-specific rules that deny the parent also bound the agent.
- **Agent and parent share post ownership.** The agent acts for its parent, so each may edit, delete, and read the other's posts under the rules for their own posts, without `edit_others_posts`. Own-post limits still apply, for example a Contributor still cannot edit published posts. Agents sharing a parent are not linked to each other.
- **Parent eligibility is a capability.** `wpai_add_agents` means "may have agents". Agents are still created by user managers; self-service creation by parents, if added, is meant to be gated by this same capability. Users without an explicit grant or denial receive it when they can `edit_posts`, and site owners restrict it per role or per user with any role editor. Agents never receive it, and they cannot provision agents for anyone else either. On multisite, a super admin can be a parent without being a member of the site.
- **Agents without an eligible parent are suspended.** When the parent is deleted, or loses `wpai_add_agents`, the agent keeps its account, content, and credentials but is denied every capability, including the self-management core otherwise allows without any, and its Application Passwords stop authenticating. An administrator restores the parent's eligibility or deletes the agent. A role change never deletes accounts, because that would delete content on a reversible administrative action.
- **Parents manage their own agents.** A parent can edit their agent's profile and issue or revoke its Application Passwords without `edit_users`. Changing the agent's role or deleting it still requires core's permissions. Whatever its role, an agent can never edit, promote, remove, or delete its own parent.
- **Agents follow their parent out.** Deleting the parent deletes their agents; agent content is reassigned to the same user as the parent's, or removed with it. Core only offers that choice when the deleted user owns content, so agent content counts toward it, and the delete screen names the agents that go too. An agent chosen to receive the content is kept, detached from the deleted parent.
- **The links are protected.** `wpai_agent` and `wpai_agent_parent` cannot be written through capability-checked meta APIs such as REST, so an agent cannot be re-parented or turned into a human account that way.
- **The role defines authority within that ceiling.** Provisioning requires `create_users`, `promote_users`, and the primitive `edit_users` capability needed to manage the resulting agent. The selected role cannot exceed the provisioner's or the parent's effective capabilities, and the provisioner comparison is repeated against the real marked account after creation so user-specific capability filters cannot widen it. Once assigned, an agent receives the same capabilities WordPress grants a human with that role, within its parent's. An Administrator agent of an Administrator is therefore fully trusted and carries the same operational risk as any other Administrator account; lower roles remain limited by WordPress's normal capability mapping.
- **Agents without administrative access cannot write unfiltered HTML.** Some roles below Administrator carry `unfiltered_html`, most notably Editor on single-site installations. Model output stored with that capability becomes stored XSS, so agents without `manage_options` pass through core's normal KSES filtering. Capability checks, rather than role names, keep custom roles aligned with core behavior.
- **Agents remain visible as users.** Hiding a principal from ordinary user queries would break ownership and capability-dependent code. The admin UI marks agents and offers an explicit filter, while normal queries continue to return them.

## Administration

Agent management stays on core user screens because the underlying resource is a user:

- **Users → Add Agent** reuses the Add User form, keeping core's identity fields, role controls, validation, and accessibility behavior. A required **Parent user** field lists the site's accounts allowed to have agents, with nothing preselected so the parent is always a deliberate choice. Password and human notification controls are omitted because nobody logs in as the account. After creation, the administrator is redirected to the agent profile to create the first Application Password.
- **Every agent username ends with `_agent`.** Provisioning appends the suffix when it is missing, including for programmatic callers. The convention makes agents recognizable wherever only the login is shown, such as WP-CLI output, author names, and logs. Accounts created outside this flow carry no such guarantee.
- **The profile** remains the canonical place for administrators to change the role, edit identity data, and issue or revoke Application Passwords. Human-only login and admin-interface preferences are hidden.
- **Human profiles** list the user's agents, so parents without access to the Users screen can reach them.
- **The Users list** labels roles such as `Editor (agent of Jane)`, or marks the agent as suspended, provides account-type filtering, and replaces the password-reset action with credential management.
- **REST user responses** expose the read-only `wpai_is_agent` and `wpai_agent_parent` fields, in every context, so clients can distinguish agent identities and render "Agent on behalf of Parent" bylines.

## Multisite

WordPress stores user identity and Application Passwords across the network, while memberships and roles are site-specific. Agent accounts follow that core model: one agent may be a member of multiple sites, and its role on each site, bounded by its parent's capabilities on that site, defines what it can do there. An agent has no authority on a site its parent does not belong to, unless the parent is a super admin. The same credential identifies the network user on every site, but it does not grant site membership or capabilities. It still authenticates the agent as a logged-in user everywhere, like any WordPress credential, so a site the agent does not belong to sees an authenticated user without capabilities there.

Agents are provisioned from a site so their initial role has site context. Adding an existing agent to another site, removing it, changing its role, and deciding who may manage it all use core's normal multisite permission and invitation flows. Removing an agent from one site removes its authority there without changing its memberships or roles elsewhere. Removing a parent from a site removes their agents from that site, and deleting the parent from the network deletes their agents. Core only lets accounts with the network-level `manage_network_users` capability edit other users on multisite, and the same rule decides who can provision agents.

**Needs discussion:** parents are a deliberate exception to that rule. They can edit their own agents' profiles and credentials on multisite without `manage_network_users`, so that per-agent credentials stay in the parent's hands on every install. This differs from the core permission model and is flagged in code for a decision with maintainers.

The only agent-specific multisite restriction is that agents cannot become super admins. Super admin is a network-wide status outside the site role system and bypasses most capability checks, so it is incompatible with role-defined agent authority.

Agent provisioning is available on multisite only when the plugin is network-activated. The agent marker is network-wide, so per-site activation cannot guarantee that every site blocks interactive login and password resets for the same account. The Add Agent UI stays unavailable and direct provisioning fails until a network administrator activates the plugin across the network. Existing-agent management remains available so credentials can still be revoked.

## Enablement and retirement

Provisioning and admin UI are loaded only when the environment supports WordPress AI and both the global AI features setting and the Agent Users experiment are enabled. The two feature settings are off by default.

Security rules for existing agents are different: they register whenever the plugin is active, before optional AI requirements and feature toggles are evaluated. Disabling the experiment hides provisioning and management enhancements but does not turn existing agents back into ordinary interactive accounts. Their login, password-reset, `unfiltered_html`, and parent restrictions, including removal with the parent, remain in force.

To retire an agent, revoke its Application Passwords or delete the account and choose how to reassign its content. Disabling the experiment does not revoke credentials or delete accounts.

Agents created before the parent link existed, or created outside the provisioning flow, have no parent and are suspended until they are recreated or deleted.

WP-CLI is intentionally outside these runtime restrictions. An operator using `wp --user=<agent>` already has shell and database authority.

## Developer reference

```php
if ( wpai_is_agent_user( $user_id ) ) {
	// Apply agent-specific presentation or behavior.
	$parent = wpai_get_agent_parent( $user_id ); // WP_User|null.
}
```

Stored metadata:

- `wpai_agent` (`Agent_Account::META_KEY`) marks the account.
- `wpai_agent_parent` (`Agent_Account::META_PARENT`) links the agent to its parent.
- `wpai_agent_created_by` (`Agent_Account::META_CREATED_BY`) records the provisioner.

`Agent_Account::PARENT_CAP` (`wpai_add_agents`) is the parent eligibility capability, and `Agent_Account::is_suspended()` reports whether an agent currently has an eligible parent.

`Agent_Account::LOGIN_SUFFIX` holds the username suffix, and `Agent_Account::apply_login_suffix()` appends it to a sanitized login when missing.

Application Passwords require HTTPS or a `local` environment type. The experiment does not override that global core requirement.

The experiment deliberately omits custom extension hooks until the identity contract is validated. It also does not cover assistants acting inside a logged-in human session, credential protocols such as OAuth, trust tiers, approval workflows, or per-run audit correlation.

## Open questions

- **Restrictions for Administrator agents.** Apart from the `unfiltered_html` rule, agents keep role parity within their parent's permissions: an Administrator agent of an Administrator can, for example, create users. Whether agents should always be denied a short list of capabilities, such as user management or plugin and theme installation, is left to maintainers.
- **Parent-controlled narrowing.** The role limits an agent within its parent's permissions, but only user managers can change it. Letting parents narrow their own agents, and create them, belongs with self-service provisioning gated by `wpai_add_agents`.
- **Multisite parent management.** See the exception above.

## Known limitations

- Every capability check for an agent also runs the same check for its parent. The cost has not been measured on large REST collections; cache the parent object, never the authorization results, if it matters.
- On the network Users screen, core offers content reassignment per site where the deleted parent is a member. Content an agent owns on a site where its parent is not a member is deleted with the agent.
- The Add Agent parent field checks eligibility for every user on the site, like core's user dropdowns.

Disable the experiment in code:

```php
add_filter( 'wpai_feature_agent-users_enabled', '__return_false' );
```

## Integrator guidance

Agent users change the answer to a question a lot of plugin code asks without noticing: given this site, which user should my code act as? Until now every account that could be resolved that way belonged to a person. Some of them no longer do, and the code that resolves one is usually old, several layers down, and written by someone who is no longer looking at it.

The rule that keeps the rest of this short: an agent is hidden from presentation, never from queries. Ownership, capability, revision, and comment code must keep seeing agents or it will draw the wrong conclusions about content they own. Author dropdowns and participant lists are where they should be filtered out.

### Choosing a user to act as

The two common shapes both resolve by authority, and authority is what an agent has:

```php
// By role.
get_users( array( 'role' => 'administrator', 'number' => 1, 'orderby' => 'ID', 'order' => 'ASC' ) );

// By capability: the usual repair when the role query proves unreliable.
foreach ( get_users( array( 'number' => 50, 'orderby' => 'ID' ) ) as $user ) {
	if ( user_can( $user, 'manage_options' ) ) {
		return $user->ID;
	}
}
```

Either can return an Administrator agent. The second one matters more, because moving a resolver from roles to capabilities is the standard advice for making it reliable, and it does not help here. An Administrator agent of an Administrator parent holds `manage_options` exactly as a human Administrator does, which is the identity model working as designed rather than a gap in it.

The two shapes also disagree about suspended agents. An agent whose parent was deleted or lost `wpai_add_agents` keeps its role but is denied every capability. The capability scan skips it; the role query still returns it, and code that then acts as that user gets an account that can do nothing, usually without an error at the point it was chosen.

Both shapes also tend to be reached for by ID order, and ID order is not a proxy for "the site owner". An agent provisioned before the humans, on a site set up agent-first by a host or by WP-CLI, sorts first. So does an account created by an attacker who backdated it, which is a pattern security scanners already look for; agent users add a legitimate account with the same sorting behaviour.

Where the resolved user will own content or stand in for a person, there are two correct answers. Exclude agents from the query:

```php
get_users( array(
	'role'       => 'administrator',
	'number'     => 1,
	'orderby'    => 'ID',
	'order'      => 'ASC',
	'meta_query' => array(
		array(
			'key'     => 'wpai_agent',
			'compare' => 'NOT EXISTS',
		),
	),
) );
```

Or, when the code already holds an agent and needs the person it acts for, step up to the parent:

```php
if ( wpai_is_agent_user( $user_id ) ) {
	$person = wpai_get_agent_parent( $user_id ); // WP_User|null: null when the parent is gone.
}
```

A parent that still exists but lost `wpai_add_agents` is returned too, while its agent is suspended; `Agent_Account::is_suspended()` covers both cases when the code needs to know whether the agent can act at all.

`wpai_is_agent_user()` and `wpai_get_agent_parent()` are the supported checks. Reading the `wpai_agent` and `wpai_agent_parent` meta directly works and will keep working, but the helpers are what survive a change in how the link is stored.

Two properties shape what a fallback chain can safely do. Agents cannot be super admins, so a resolver that falls back to `get_super_admins()` cannot reach one that way. And `get_users( array( 'role' => 'administrator' ) )` is scoped to a site's own member list, so on multisite it already misses a network administrator who runs a subsite without being added to it: that resolver was returning empty on some installs before agent users existed, and the repair for it is usually the capability scan above, which reintroduces the agent case.

A stored user ID needs one more thought. Agents follow their parent out: deleting the parent deletes its agents. Code that saved an agent's ID as "the account to act as" will find it gone after a routine user removal, so where the choice should outlive staff changes, store the parent, or resolve again each time.

### Attribution and display

`post_author` can now point at an agent. Code that treats an author as a person will ask an agent for a biography, an avatar, an email to notify, or an "is this author still with us" answer, and get something plausible and wrong. The parent link gives each of those a better answer:

- Bylines and author archives can show the agent on behalf of its parent. REST responses carry `wpai_is_agent` and `wpai_agent_parent` in every context for this, so a client does not need a second request. Whether an agent's name reaches visitors at all is still an editorial decision for the site, not a default a plugin should make on its behalf.
- Notification and digest code that mails the author should mail the parent. An agent account has an address and no reader; the parent is the person who answers for the work.
- "Written by a human" or "needs a human review" logic should treat agent-authored content as the parent's responsibility rather than loop, or decide it has no owner.

Attributing work to an agent is the point of the feature. The adjustment is not to hide the attribution, it is to stop inferring a person from it, and to ask the parent when a person is what the code needs.

### Content written by an agent

Agents without `manage_options` do not hold `unfiltered_html`, so their content passes through KSES like any other filtered write. Code that renders agent-authored content should expect it to have been filtered, and code that stores it should not assume markup survives verbatim.

This matters most for anything that writes structured markup: page-builder payloads, embeds, and scoped styles are the shapes KSES alters or drops. An agent that produced valid builder output on a site where it held `unfiltered_html` can produce broken output on a site where it does not, with no error at the point of writing. The safe set of tags is a property of what a particular site has installed, so it can only be decided by the site.

The parent ceiling adds a second way for the same agent to behave differently over time. Every capability check also runs against the parent, with the same arguments, so demoting the parent or an object-level rule that denies the parent narrows the agent at once. A write that worked yesterday can be refused today without anything changing on the agent itself; report the refusal rather than retrying under another account.

### What not to change

Some defensive instincts make this worse:

- Do not filter agents out of `get_users()` globally, through `pre_get_users` or otherwise. Ownership and capability code depends on seeing them, and a hidden principal produces failures that are much harder to trace than a visible one.
- Do not key behaviour on the `_agent` username suffix. It is a readability convention applied at provisioning, not a guarantee: accounts created outside that flow do not carry it, and a login is not an identity contract.
- Do not cache authorization results for an agent. The answer depends on the parent's current permissions; cache the parent object if the cost matters, never the outcome of a capability check.
- Do not treat "is an agent" as "is untrusted". The role decides authority within the parent's permissions. An Administrator agent of an Administrator is fully trusted and carries the operational risk of any Administrator account.
