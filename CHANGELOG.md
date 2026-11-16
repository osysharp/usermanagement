# Osysharp.UserManagement — changes

## 0.13.0
- The People page's head is a band across the whole area (the page-head tokens' ground and rule), its words in the page's own
  measure; the list below keeps `Length.PeoplePageW` (no longer a hard-coded 1120px — the token is the theme's to set). The
  API keys and Access changes pages draw their `SettingsPage` head as the same band.
- A table's title on the API keys page is sized by `FontSize.PageSectionTitle`, so a shell with a small page title sets its sections' headings smaller.
- Needs Osysharp.Ui 3.5.0.
