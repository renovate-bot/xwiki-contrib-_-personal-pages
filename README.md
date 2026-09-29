# Personal Pages Application

Personal pages for every XWiki user, along the lines of the
[Personal Pages design proposal](https://design.xwiki.org/xwiki/bin/view/Proposal/PersonalPages).
Users create their own pages with one click; administrators choose where they are created and who can read them.

* Project Lead: [Karsten Sandmand](https://www.xwiki.org/xwiki/bin/view/XWiki/KiloNiner)
* [Documentation & Download](https://extensions.xwiki.org/xwiki/bin/view/Extension/Personal%20Pages%20Application/) (after publication)
* [Issue Tracker](https://jira.xwiki.org/browse/PERSONALPG)
* Communication: [Forum](https://forum.xwiki.org/), [Chat](https://dev.xwiki.org/xwiki/bin/view/Community/Chat)
* [Development Practices](https://dev.xwiki.org)
* Minimal XWiki version supported: XWiki 16.10.0
* License: LGPL 2.1
* Translations: N/A
* Sonar Dashboard: N/A
* Continuous Integration Status: [![Build Status](https://ci.xwiki.org/job/XWiki%20Contrib/job/personal-pages/job/master/badge/icon)](https://ci.xwiki.org/job/XWiki%20Contrib/job/personal-pages/job/master/)

## Features

* **Create my personal page** button on `PersonalPages.WebHome`, a **My pages** entry in the drawer menu, and a
  **Personal Pages** entry in the Applications panel.
* **All personal pages** table on the hub (searchable, sortable, paged; rows filtered by the viewer's rights). It lists
  personal pages by their owner marker, so pages created under an earlier location are included.
* **Personal pages** section in every user profile, linking to that user's pages (when the viewer may see them).
* **Administration → Users & Rights → Personal Pages**:
  * *Create personal pages under*: any page of the wiki (default: `PersonalPages.WebHome`).
  * *Who can read new personal pages*: all logged-in users (default), only the owner, or the owner and chosen groups.
  * *What the owner can do*: edit, comment, delete, administer (default: edit and delete).
  * *Create missing personal pages*: creates personal pages for every active user of the wiki who has none, in
    batches of 50 that continue automatically. Pages created this way have the administrator as creator.
* Each user's pages are titled with their name. To add a suffix (for example when personal pages live under a
  department page), change the `personalpages.page.title` translation, e.g. to `{0}''s pages` (`''` produces an apostrophe).
* Settings apply to personal pages created afterwards. Existing pages keep their location and permissions, and the
  **My pages** link keeps finding them.
* Works in subwikis, including for users of the main wiki (rules and markers use wiki-prefixed user references when
  needed, and "all logged-in users" also covers the main wiki's `XWikiAllGroup`). The bulk action covers the
  subwiki's local users.

## How it works

1. The user clicks **Create my personal page** (POST with a CSRF form token).
2. `<location>.<username>.WebPreferences` is saved **as the author of the hub page** (an administrator). It grants
   the owner the configured rights and grants read access explicitly (to `XWikiAllGroup`, the chosen groups, or
   only the owner), so personal pages never depend on the permissions of the location page.
3. `<location>.<username>.WebHome` is then saved **as the user**, with a `PersonalPages.Code.PersonalPageClass`
   marker object. A user's pages are found by that marker, and only count when the page's creator is the user or an
   administrator, and the page sits in a space named after the user (where personal pages are always created).
   Users can edit the marker on their own pages but cannot change who created a page or where it lives, so nobody can
   claim someone else's pages, and moving the location later does not break anything.

The shared logic lives in `PersonalPages.Code.Macros` (Velocity macros included by the hub, the drawer entry and the
profile section). The bulk action runs on the hub page too, because administration section code cannot use macros
from included pages.

The settings are stored on `PersonalPages.Code.Configuration`, edited with XWiki's standard administration form
(`XWiki.ConfigurableClass`). The page is readable by administrators only, and it is declared as a `configuration`
XAR entry in the POM: the Extension Manager installs it with the default settings and never touches it again on
upgrades, so administrators keep their settings. (A manual re-import through Administration → Import ignores entry
types and resets the settings to the defaults.)

The shared logic lives in `PersonalPages.Code.Macros` (Velocity macros included by the hub, the drawer entry and the
profile section). The bulk action runs on the hub page too, because administration section code cannot use macros
from included pages.

### Implementation notes

* Rights levels are stored as `view,edit` (no spaces): XWiki's rights checker does not trim them, so a list saved
  as `view, edit` only grants its first right.
* The hub page must be saved by a user with script and admin rights, which is the case when an administrator
  installs the extension.
* XWiki's built-in password login does not accept usernames containing `.`, `@` or spaces; the LDAP and OpenID Connect
  authenticators clean such characters out of the XWiki user names they create. Personal pages themselves handle such
  names (tested via the bulk action).

## Building

```
mvn clean install
```

Without a local Java/Maven, the build can run in Docker:

```
docker run --rm -u "$(id -u):$(id -g)" -e MAVEN_CONFIG=/var/maven/.m2 \
  -v "$PWD":/src -v "$HOME/.cache/xwiki-m2":/var/maven/.m2 -w /src \
  maven:3-eclipse-temurin-21 mvn -B -Duser.home=/var/maven -s settings.xml clean install
```

The `.xar` ends up in `target/`.
