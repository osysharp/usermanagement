# Osysharp.UserManagement

The **user manager**: who is in the app, what each of them may do, and who changed that — when, and why.

- A **people list** — search by name or address; each person's roles (by the names the app gives them), whether two-step
  sign-in is on, whether their address is proven, when they last signed in; deactivated people kept and marked.
- A **person in full** — their app roles and the roles they hold in organisations, where else they are signed in right
  now, and the history of every change to their access (and the invitation they joined by).
- **Changes**, each one a record first: grant and revoke the app's roles · deactivate and reactivate · reset a lost
  second factor · sign out everywhere · add to and remove from an organisation, and grant roles within one.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.UserAccounts@1;
use Osysharp.Organizations@0;
use Osysharp.Invitations@2;
use Osysharp.UserManagement@0;
```

```osy
using Osysharp.Identity;
using Osysharp.UserManagement;

[Role] enum AppRole { Authenticator, Agent, Lead, Admin, Integration }
policy IsAdmin => RoleGrant.Any(g => g.Member == user && g.Role == AppRole.Admin);

policy ManagesPeople => IsAdmin;            // who changes people's access
policy MayInvite => IsAdmin;                // Osysharp.Invitations
policy ManagesOrganizations => IsAdmin;     // Osysharp.Organizations

// The roles a manager may hand out, by the names people read. A role not listed is never offered — `Integration` here.
app.Roles = new RolesSetup { Offered = [
  new RoleChoice { Role = AppRole.Agent, Label = "Agent", Description = "Works tickets." },
  new RoleChoice { Role = AppRole.Lead, Label = "Lead", Description = "Watches the queues." },
  new RoleChoice { Role = AppRole.Admin, Label = "Admin", Description = "Runs the desk." },
] };

// A role held only inside an organisation is given only there — `HeldIn = RoleScope.Organization` on its RoleChoice.

// The role the app must never be left without an active holder of.
app.People = new PeopleSetup { KeepOneOf = AppRole.Admin };

// The user manager READS who holds which role — every change goes through it, so grant nothing more here.
partial entity RoleGrant { security { allow read when ManagesPeople; } }
```

Then put `PeopleManager()` on a page only managers reach (it is a `[Composable]`; it brings the invite form and the
waiting invitations with it).

Who is offered the invite is `MayInvite`, which the manager asks the SERVER once (`MayInvitePeople()`) — so it may read
any table, e.g. a role's permissions kept as rows:
`policy MayInvite => RoleGrant.Any(g => g.Member == user && RolePermission.Any(p => p.Role == g.Role && p.Permission == Permission.Invite));`.
Write `PeopleManager()` — not `PeopleManager(canInvite: MayInvite)`, which hands the policy to a control and so allows
only a check on the caller's own rows. `canInvite: false` leaves inviting out; it never widens what `MayInvite` says. The
invite itself is refused by the Invitations kit's rules whatever a page offered.

In a person's drawer, **Add role** lists every role the app offers, with the ones they hold marked *Held*. What takes
something away — *Remove <role>* and *Deactivate* — sits in one danger area at the foot of the drawer: each says what
follows in one line, both are the same quiet button, and each asks first.

## Where the pages sit in your settings

The kit announces three settings pages: **People** and **API keys** under "Team", and **Access changes** under
"Security" (it is an auditor's page). An app with groups of its own places any of them itself, by route:

```
class MyPlacement : ISettingsPlacement {
  public SettingsPlacement[] Placements() {
    return [new SettingsPlacement { To = "/settings/access", Group = "Audit", Order = 40 }];
  }
}
```

The pages are drawn with the Ui kit's theme tokens only, so they take your theme's accent and type scale, and its
density: `Space { Step = "0.2rem"; }` with `Length { ControlSm = "28px"; ControlMd = "32px"; }` tightens every page in
the app, these included.

From code: `GrantRole(p, AppRole.Lead, "covering December")`, `RevokeRole`, `Deactivate(p, "left")`, `Reactivate`,
`ResetSecondFactor(p, "lost phone — confirmed by a call")`, `SignOutEverywhere(p)`, `AddToOrganization(org, p)`,
`GrantInOrganization(org, p, AppRole.Lead)`; reads `PeopleList(search)`, `PersonDetailOf(id)`, `AccessTrail()`.

## API keys

A people manager issues a teammate an API key on `/settings/api-keys` (or `ApiKeyManager()` on a page of your own), so
that a program can call the app's REST API as that teammate, with exactly what they may do. The key is shown **once**,
with a copy button, and only its hash is kept on the person's account (two at a time, so a key can be replaced with no
gap). Under the heading *Keys in use* a table says, for each key, its name, how it ends, whom it acts as, who issued
it and when, and when it was last used, with Revoke at the end of the row. Revoking one, or
deactivating its holder, refuses it from the next request. Issuing and revoking are access changes like any other,
on the trail with who did it and why. From code: `IssueApiKey(p, "Nightly export").Key`, `RevokeApiKey(key)`.

| Shown once, after it is issued | Revoked, in the dark |
|---|---|
| ![a new API key shown once with a copy button](docs/api-keys-once.png) | ![the revoked keys list in the dark theme](docs/api-keys-revoked-dark.png) |

## Integrations

Zapier, a nightly export, a partner's system: a program that works with your app should do it under its own name, not
a teammate's. An **integration** is an account of its own — a name, the role it acts with, the person who answers for
it, and API keys — with no mailbox and no password, and no way to sign in. Everything it does through the API is
recorded as its own ("Zapier added a note"), so the trail never blames a person for a program.

A people manager adds one on the same page — a name and what it may do — and is shown its first key once. It is listed
with its role, its owner and how many keys it holds; **Switch off** deactivates it, and every key it holds is refused
from the next request. Adding one is an access change on the trail, like any other. From code:
`AddIntegration("Zapier", AppRole.Agent)` then `IssueApiKey(zapier, "Web form").Key`.

An integration can also work for **one organisation** only — an agent key for one customer, one team, one product.
`AddIntegration("Acme feed", AppRole.Agent, organization: acme)` gives it the role inside Acme and no role across the
app, so its key reads what Acme's members read and nothing of any other organisation. A role is given only where the
app says it is held (`RoleChoice.HeldIn`): an app-wide role is refused inside an organisation, and an organisation's
role is refused without one. The trail reads *Added the integration “Acme feed” as Agent at Acme*.

An integration can be added **holding no role**: `AddIntegration("Relay")`. It is the account, its owner and the record
of its creation, and nothing else — its key is accepted and can do nothing that needs a role. This is for an app whose
own rules give a role only through the app's own function: add the integration with no role, then give it one by that
function. The pages say *Holds no role* until then, and a people manager who may gives it one from its drawer on the
people page. The trail reads *Added the integration “Relay” with no role, owned by Sam*.

An app that lists keys and integrations on a page of its own draws one record a row: `ApiKeyRow(line, onRevoke)` over
`ApiKeyLines(now)` and `IntegrationRow(line, onSwitchOff, onReassign)` over `IntegrationLines()`, each in a
`Stack(role: UiRole.List)`. The kit's own page lists them in tables (`RecordTable`, `RecordRow`), which is what a new
page should use.

An integration outlives the person who answers for it. When they leave and their account is deactivated, the program
keeps working, and its row says **Owner deactivated**. **Reassign** gives it to somebody who is here, with a reason, as
an access change on the trail like any other. From code: `ReassignIntegration(zapier, sam, "Ari left the team")`.

| Its owner has left | Given to somebody who is here |
|---|---|
| ![an integration marked owner deactivated, still holding its key](docs/integrations-owner-left.png) | ![choosing the integration's new owner, with a reason](docs/integrations-reassign.png) |

| Adding one | Listed with its role, owner and keys |
|---|---|
| ![adding an integration: its name and what it may do](docs/integrations-add.png) | ![an integration listed beside the key it holds](docs/integrations-list.png) |

## Every change is a record first

A manager never writes somebody's access directly. Each change is an `AccessChange` — who made it, to whom, what, when
and why — and its run applies it **in the same commit**. Only the app's people managers may make one, naming
themselves; nobody edits or deletes one. So the trail of who changed whose access is not a log that might be missing a
path: it is the only path.

The run checks every change before it applies it — a record made without the functions' own checks is **refused**, with
the reason, and nothing changes:

- a role is granted only if the app **offers** it (`app.Roles`) — never `Authenticator`, never an integration role;
- a manager never deactivates **themself**, and the app is never left without an **active** holder of `KeepOneOf`;
- a second factor is reset only with a **reason** written down;
- deactivating, resetting a second factor and signing out everywhere end every session the person has, on every server,
  and each ended session is told an administrator changed the account's access — not that its password changed.

Resetting a second factor clears the authenticator, the recovery codes and a texted step, mails the person, and signs them
out: they sign in with their password and set it up again — or, where the app requires it, the sign-in asks them to.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.Organizations;
using Osysharp.UserManagement;
using Osysharp.Mail;

[Test(Team)]
void a_manager_grants_and_revokes_an_offered_role_and_each_change_is_recorded() {
  var mo = Person("mo@desk.test");
  runas (Sam) { GrantRole(mo, AppRole.Lead, "Covering the queue in December"); }
  runas (Mo) { Assert.True(RoleGrant.Any(g => g.Member == Mo && g.Role == AppRole.Lead)); }
  runas (Sam) { RevokeRole(mo, AppRole.Lead); }
  runas (Mo) { Assert.False(RoleGrant.Any(g => g.Member == Mo && g.Role == AppRole.Lead)); }
  runas (Sam) {
    var trail = AccessChange.Where(c => c.Person == mo).OrderBy(c => c.At).ToList();
    Assert.Equal(2, trail.Count);
    Assert.Equal(AccessChangeKind.RoleGranted, trail[0].Kind);
    Assert.Equal("sam@desk.test", trail[0].By.Email);
    Assert.Equal("Covering the queue in December", trail[0].Reason);
    Assert.Equal(AccessChangeKind.RoleRevoked, trail[1].Kind);
  }
}
```

From [`tests/access.test.osy`](tests/access.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.UserManagement;

[Test(Team)]
void a_key_acts_as_the_person_it_was_issued_to_and_a_revoked_key_is_refused_at_once() {
  var ada = Person("ada@desk.test");
  IssuedApiKey issued = new IssuedApiKey();
  runas (Sam) { issued = IssueApiKey(ada, "Nightly export"); }
  Assert.StartsWith("pk_", issued.Key);
  // The key is Ada — not Sam, who issued it.
  Assert.Contains("ada@desk.test", Me(issued.Key));
  Assert.DoesNotContain("sam@desk.test", Me(issued.Key));
  runas (Sam) {
    var k = ApiKey.Single(x => x.Id == issued.Id);
    Assert.Equal(1, k.Slot);
    Assert.Equal("…" + issued.Key.Substring(issued.Key.Length - 4), k.Ending);
    // Its use is recorded, to the minute.
    Assert.NotNull(User.Single(u => u.Email == "ada@desk.test").ApiKeyUsedAt);
    RevokeApiKey(k, "Export moved to the warehouse");
  }
  // Refused at the very next request.
  Assert.Equal("HTTP 401", Me(issued.Key));
  runas (Sam) {
    var trail = AccessChange.Where(c => c.Person == ada).OrderBy(c => c.At).ToList();
    Assert.Equal(AccessChangeKind.ApiKeyIssued, trail[0].Kind);
    Assert.Equal(AccessChangeKind.ApiKeyRevoked, trail[1].Kind);
    Assert.Equal("Export moved to the warehouse", trail[1].Reason);
    Assert.Contains("Revoked the API key “Nightly export”", AccessTrail()[0].What);
  }
}
```

From [`tests/api-keys.test.osy`](tests/api-keys.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.Organizations;
using Osysharp.Invitations;
using Osysharp.UserManagement;
using Osysharp.Mail;
using Osysharp.Workflow;

[Test(RuledDesk)]
void an_integration_is_never_given_a_role_the_apps_rule_keeps_for_people() {
  runas (Sam) {
    var bot = AddIntegration("Export bot", AppRole.Agent, reason: "Nightly export");
    // Refused AT ONCE, to the manager who asked, in the app's own words — not a run that fails later.
    var refusal = Assert.Throws(() => GrantRole(bot, AppRole.Auditor, "Reads the books"));
    Assert.Contains("Auditor is a person's role — an integration is never given it.", refusal.Message);
    // …and the trail records the change as refused, with that reason.
    var refused = AccessChange.Single(c => c.Person == bot && c.Role == AppRole.Auditor);
    Assert.Equal(AccessChangeState.Refused, refused.State);
    Assert.Contains("an integration is never given it", refused.Refusal ?? "");
    // …and a person is given it by the same route.
    GrantRole(Account("mo@desk.test"), AppRole.Auditor, "Joins the audit");
  }
  Assert.Equal(1, Holders(AppRole.Auditor));
  runas (Sam) { Assert.True(RoleGrant.Any(g => g.Role == AppRole.Auditor && g.Member.Email == "mo@desk.test")); }
}
```

From [`tests/app-rules.test.osy`](tests/app-rules.test.osy).

## Learn more

- [The Osysharp.UserManagement reference](https://osysharp.com/reference/security/people-kit/)
- [Osysharp.UserManagement in the package catalogue](https://osysharp.com/packages/#osysharp-usermanagement)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
