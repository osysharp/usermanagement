# Osysharp.UserManagement — changes

## 0.15.0
- The hourly sweep reaches a change that has NO run (one recorded before `AccessChangeFlow` was deployed): the workflow names
  who may start it by hand, `StartedBy = u => u == Actor.Of<AccessChanger>()`. With the default door ("may change the row")
  the sweep's `StartAgain` was refused for such a change, which then waited for ever.
- A change to somebody's access is made when it is asked for, or not at all: one its run did not apply within 24 hours
  (`AccessChangeWaitsAtMost()`) is expired — the new state `AccessChangeState.Expired`, appended after `Refused` — with the
  reason, and never made. Its run checks the age first, before anything else, and an hourly sweep (`AccessChangeExpiry`, on
  the one `AccessChangeSweep` row the app's start hook makes) starts the run again of every change still waiting after its
  day, so a change whose run failed or ended is expired too. The manager who asked is mailed (`AccessChangeExpiredMail`);
  the trail marks it "Expired" (`AccessEvent.Expired`), and a function that recorded it says why it was not made.
- `AccessChange.At` is a `[CreatedAt]` stamp, written by the platform when the change is recorded: a change's age is not
  the caller's to set. Rows recorded before keep the time they were given — and one that says it was asked for more than
  five minutes ahead of the clock (`AccessChangeClockSkewAtMost()`) is expired as one that waited too long.
- The `AccessChangeSweep` row is made by the start hook as `AccessChanger` (`[OnStart(RunsAs = AccessChanger)]`); a people
  manager no longer may create one.

## 0.13.0
- The People list (and who a key can be issued to) leaves out every declared actor's account — a workflow's run, an agent,
  this kit's `AccessChanger`, Invitations' `InvitationRun`, the app's own robots — whatever the directory's `Listed`/`Hidden`
  say. Which accounts are actors is the platform's catalogue (`Actor.PrincipalIds()`), never an address. 0.12.0 listed
  them as people.
- The People page's head is a band across the whole area (the page-head tokens' ground and rule), its words in the page's own
  measure; the list below keeps `Length.PeoplePageW` (no longer a hard-coded 1120px — the token is the theme's to set). The
  API keys and Access changes pages draw their `SettingsPage` head as the same band.
- A table's title on the API keys page is sized by `FontSize.PageSectionTitle`, so a shell with a small page title sets its sections' headings smaller.
- Needs Osysharp.Ui 3.5.0.
