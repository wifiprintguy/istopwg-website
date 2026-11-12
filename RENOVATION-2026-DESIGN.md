# PWG Website Renovation 2026 Design Document

This GitHub project contains the static source code for the PWG's website (www.pwg.org). The PWG Steering Committee and PWG infrastructure administrators have known for some time that the PWG needs to update the site to satisfy some unsatisfied requirements.

## Current Site

### Overview

The site is a hybrid: mostly hand-written static HTML, plus a small custom PHP application called **dynamo** that applies a common page template and runs the database-backed features (accounts, document registry, news articles, issue tracking). It runs on a traditional LAMP stack:

- **Host:** a single Linode Linux virtual machine, which according to the PWG Communications Policy [PWG-COMM-POLICY] also hosts the FTP site, role email addresses and the Mailman mailing list server
- **Operating system and versions:** *Observed in November 2026* from the site's `Server` and `X-Powered-By` headers: CentOS 7, Apache 2.4.6, OpenSSL 1.0.2k and **PHP 5.4.16**. PHP 5.4 stopped receiving security fixes in September 2015, and CentOS 7 reached end of life in June 2024, so neither the operating system nor PHP has had security updates for some time.
- **Web server:** Apache 2.4 with `mod_actions`, `mod_php` and `mod_ssl`
- **Application:** PHP (no framework, no Composer dependencies), about 9,700 lines across the `dynamo/` directory
- **Database:** MariaDB/MySQL, accessed through PDO
- **Front end:** Bootstrap 3.3.7 and jQuery 3.2.1 loaded from public CDNs (older Bootstrap 3.2.0 copies are also kept in `dynamo/resources/`), plus a small amount of site-specific CSS and JavaScript
- **Search:** Google Programmable Search (Custom Search Engine) embedded in a modal dialog
- **Document archive:** `ftp.pwg.org/pub/pwg/...`, the PWG's long-standing archive (about 20 years) of specifications, drafts, minutes, charters and other documents. Despite the name, files are usually served over HTTP/HTTPS, and `ftp.pwg.org` URLs are cited widely in PWG specifications, IANA registrations, mailing list archives and elsewhere.

The source is hosted on GitHub at `istopwg/website`. There is no build step, CI pipeline or automated test suite.

### Repository Layout

The repository is organized by workgroup or topic:

| Path | Contents |
|------|----------|
| `/` (root) | Top-level pages: `about.html`, `members.html`, `standards.html`, `informational.html`, `namespaces.html`, `bofs.html`, `mailhelp.html`, `search.html`, plus `robots.txt`, `.htaccess` and the touch icon |
| `3d/` | 3D printing (IPP 3D) information and infographics |
| `blog/` | About 45 static HTML articles, mostly face-to-face meeting summaries and press releases (2013–2021) |
| `chair/` | Officers page, participation information, `meeting-info/` (historical meeting logistics pages, maps, agendas) and `membership_docs/` |
| `cloud/`, `sm/`, `wims/` | Pages for inactive workgroups (Cloud Imaging Model, Semantic Model, WIMS) |
| `ids/` | Imaging Device Security workgroup |
| `ipp/` | IPP workgroup: overview, FAQs, IPP Everywhere, the IPP Guide (HTML/PDF/EPUB), and the IPP registry |
| `ippeveselfcert/` | IPP Everywhere self-certification information |
| `printers/` | IPP Everywhere printer list |
| `security/` | Security information pages |
| `pwg-logos/` | Member company logos |
| `dynamo/` | The PHP application (see below) |

The whole repository is about 100 MB and holds about 1,500 files, many of which are PDFs and images.

### Request Handling and Page Templating

Content pages are plain HTML fragments without site chrome. Apache is configured (`dynamo/config/pwg-apache.conf`) with:

```
Action text/html /dynamo/wrap.php
```

so **every `.html` request is routed through `dynamo/wrap.php`**, which:

1. Rejects paths containing `..` or naming files that don't exist (returns 404).
2. Sends three special pages to PHP scripts: `/index.html` → `index.php`, `/standards.html` → `standards.php`, `/informational.html` → `informational.php`. The `.html` files in the repository for these pages are placeholders.
3. Otherwise reads the HTML file and extracts:
   - the `<title>` text (with "- Printer Working Group" removed),
   - an optional subtitle from a `<!--subtitle ... -->` comment,
   - the first inline `<style>` block,
   - the body, from `<body>…</body>` or from a legacy `<div id="PWGContentBody">` wrapper.
4. Writes the standard header (`site_header()` in `dynamo/phplib/site.php`), an `<h1>` if the page has none, the extracted body, and the standard footer (`site_footer()`).

`site_header()` generates the whole navigation bar from hard-coded PHP strings: Home, About, Our Work, Workgroups and Search menus, plus a user menu whose items depend on the logged-in user's roles. The footer holds the copyright and trademark notice and loads jQuery, Bootstrap, `pwg.js` and `pwg-cookie-notice.js`. **Changing site navigation therefore means editing PHP code.**

Some pages skip the wrapper and are complete standalone HTML documents that load their own CSS and scripts (for example `printers/index.html`).

`.htaccess` adds redirects for `/issues` and `/issues/N` to `dynamo/issues.php`.

### Dynamic Features (dynamo)

`dynamo/phplib/` holds the shared library code:

- `site.php`: loads configuration, sets up legacy globals (`PATH_INFO`, `PHP_SELF` and so on), and provides the header and footer
- `auth.php`: cookie-based sessions (`PWGSID` cookie with user ID, timestamp and HMAC; sessions last one day) and the role flags `LOGIN_IS_ADMIN`, `LOGIN_IS_EDITOR`, `LOGIN_IS_MEMBER` and `LOGIN_IS_OFFICER`
- `db.php`: a thin PDO helper layer (`db_create`, `db_load`, `db_save`, `db_search`, and others)
- `db-*.php`: model classes for each table, including their HTML form and view rendering
- `html.php`: HTML form and formatting helpers (the largest file, about 1,700 lines)
- `validate.php`, `plist.php`: input validation and Apple plist parsing (the plist parser was used for self-certification results)

The top-level `dynamo/*.php` scripts are the pages:

| Feature | Scripts | Notes |
|---------|---------|-------|
| Home page | `index.php` | Jumbotron, collapsible "display until" news alerts, and technology panels; news comes from the `article` table |
| Standards and informational lists | `standards.php`, `informational.php` | Generated from the `document` table and grouped by PWG 51xx series; series names are hard-coded |
| Document management | `documents.php` | Editors and officers create document records and upload files; uploads go to the FTP server under `/pub/pwg/<workgroup>/{wd,minutes,white,charter}/` |
| News articles | `articles.php` | Officers and admins manage home-page news |
| Issue tracking | `issues.php` | Errata and document update requests, with comments |
| Accounts | `login.php`, `logout.php`, `newaccount.php`, `forgot.php`, `enable.php`, `account.php`, `accounts.php`, `request.php` | Self-registration and password reset (both behind Google reCAPTCHA), admin user management, access requests sent by email |
| Organizations and workgroups | `organizations.php`, `workgroups.php` | Admin-only management; each workgroup record includes its FTP directory |
| IPP Everywhere | `eveprinters.php`, `evesubmit.php` | Now just redirects to `/printers/` and `/ippeveselfcert/` |
| Deployment hook | `github.php` | GitHub push webhook (HMAC-SHA1 checked) that runs `git pull` in the site root and emails the result to the webmaster |

The database schema is in `dynamo/config/pwg.sql` (tables `user`, `organization`, `workgroup`, `article`, `document`, `issue` and `comment`) and `dynamo/config/selfcert.sql` (`printer`). Test data is in `test.sql`, and `loadtest.sh` builds a local test database.

Local, secret configuration lives in `dynamo/config/site.cfg`, which is created from `site.cfg.tmpl` and kept out of Git. It holds site settings, the session secret, reCAPTCHA keys, database credentials and FTP credentials. PHP needs `short_open_tag = On`, because much of the code uses `<?` tags, and a 32 MB upload limit (`pwg-php.ini`).

### Data-Driven Static Content

Several "dynamic-looking" pages are actually static files that are edited by hand and committed to Git:

- **IPP Everywhere printer list:** `printers/printers.json` (about 2,000 models) is fetched and filtered in the browser by script in `printers/index.html`. A PWG volunteer adds new models by hand-editing and committing the JSON file. From about 2015 to April 2020, dynamo had a submission form (`evesubmit.php`). Logged-in members chose their organization, entered product family, models, URL and contact details, picked the Self-Certification Manual version, and uploaded the three self-certification result files (Bonjour, IPP and document tests, as Apple plists), which `printer_validate_plist()` checked automatically. Approved printers went into the `printer` table, and `evenotify.sh` emailed a weekly summary to `pwg-announce`. In April 2020 (commit `cb42737`) this was replaced by redirects to the static pages. The static JSON list was introduced in anticipation of moving the site to Jekyll and GitHub Pages, a move that never happened. The interim arrangement has continued since. Submitters have repeatedly asked for a capability the static list cannot offer: **private submissions that stay hidden until a vendor-chosen "turn on" (release) date**, so certification can be completed before a product is announced. *(The release-date field does not appear in this repository's history of `selfcert.sql` or `evesubmit.php`; how it was implemented should be confirmed.)*
- **IPP registry:** `ipp/ipp-registrations.xml` is an IANA-style registry rendered in the browser through `ipp-registrations.xsl`, with a RELAX NG schema (`ipp-registrations.rng`).
- **Blog and meeting pages:** each article or meeting is an individual hand-authored HTML file; there is no index generation, tagging or feed.

### News and Blog

News and blog posting work as one two-part system:

- **News items** are database records (`article` table), created by officers and admins in `dynamo/articles.php`. Each has a title, a short summary (which supports a small wiki-style link markup, `[[url|text]]`), a link, an optional workgroup, and an optional **display-until date**:
  - While the display-until date is in the future, the item is pinned as a collapsible alert at the top of the home page. This is used for open Last Calls, Formal Votes and upcoming meetings.
  - Otherwise the item appears in the home page's "PWG News" list, which shows the five most recent items. Older items are listed by `articles.php`.
- **The link points to the full story**, which lives elsewhere:
  - a hand-written HTML page in `blog/` (face-to-face meeting summaries, press releases, articles);
  - a `pwg-announce` mailing list archive post (Last Call and Formal Vote announcements);
  - a document on the archive (newly approved standards).

So publishing a typical news item takes several steps in different systems: write and commit a blog page (or send a `pwg-announce` email), then log in to dynamo and create the news item that links to it. There is no RSS/Atom feed. The most recent `blog/` page is from August 2021.

### Deployment and Hosting

- Production runs the configuration described above, but the Apache config notes that it differs from the repository layout: "some static content is hosted in the FTP directory and the dynamo content is aliased in."
- According to the README, pushes to `dynamo/` reach the site through the GitHub webhook (`github.php` → `git pull`). Static content outside `dynamo/` is served from the FTP server's `pub/pwg/www` directory, so it must also be synchronized there to go live. [PWG-COMM-POLICY] also gives the webroot as `/pub/pwg/www`. *(How changes get from GitHub into that directory today should be confirmed with the infrastructure administrators.)*
- **FTP is the only way to change the live static content.** The Linode site gives contributors no SSH access, so static pages reach the site only by FTP upload to `pub/pwg/www`, separately from the Git repository. *Observed in November 2026:*
  - The FTP server offers encryption (`AUTH SSL`), but the TLS handshake fails with current clients (`dh key too small`): the server's Diffie-Hellman parameters are too weak for modern OpenSSL. In practice, uploads and the FTP password travel unencrypted.
  - The live web root holds **28 top-level entries that are not in the repository**, all still served. They include old workgroup folders (`cam/`, `fin/`, `jmp/`, `mfd/`, `p1394/`, `ps/`, `qualdocs/`, `sdp/`, `sense/`, `updf/`, `wbmm/`, `xhtml-print/`), `mib/`, `schemas/`, `hypermail/`, `press-releases/`, `archive/`, `calls/`, `cr/`, `images/`, `cgi-bin/`, the placeholder `index.html` and a few loose files. The repository is therefore a subset of the live site. A move to a new host, or any automated sync, must keep these (or deliberately redirect or retire them), and must never delete files that are missing from Git.
- Specifications, drafts, minutes and charters are not in this repository. They live on `ftp.pwg.org` and are linked by URL from the `document` records and static pages. Because those URLs have been cited for about 20 years, the `ftp.pwg.org` archive and its URL structure are a long-term fixture, not a candidate for replacement. As the 2021 draft of the Communications Policy notes [PWG-COMM-POLICY-DRAFT], many older specifications embed `ftp://` URLs to the site, so it is "risky to 'clean up' anything" there. Newer documents cite the same paths as `https://ftp.pwg.org/pub/pwg/...` URLs (the PWG Document Management Policy [PWG-DOC-MGMT-POLICY] uses that form throughout), and dynamo rewrites `ftp://ftp.pwg.org/` links to `https://` when a document record is saved. So the path structure under `/pub/pwg/` is what has to be kept on any future host, for both schemes.
- Mailing lists are handled by a separate Mailman installation (`/mailman` is excluded in `robots.txt`; `update-mailman.sh` and `update-archives.sh` are maintenance helpers).
- HTTP usage statistics come from AWStats (`/awstats/awstats.pl`), which officers and admins reach from the user menu.

### Third-Party Dependencies at Runtime

- `maxcdn.bootstrapcdn.com`: Bootstrap 3.3.7 CSS and JavaScript
- `code.jquery.com`: jQuery 3.2.1 (3.4.1 slim on the printers page)
- `cse.google.com` / `www.google.com/cse`: site search
- `www.google.com/jsapi`: legacy Google loader
- Google reCAPTCHA on account forms
- `ieee-isto.org`: privacy policy link

### Standards Development Workflow

The site supports, but does not host, the PWG's standards development process:

- **Specifications are written in Microsoft Word.** Working drafts are posted to the archive in each workgroup's `wd/` folder (for example `ftp.pwg.org/pub/pwg/ipp/wd/`), usually as an editable `.docx`, a clean PDF and a redline PDF (dynamo's `editable_url`, `clean_url` and `redline_url` fields). Approved standards go to `/pub/pwg/standards/`. The dynamo document registry records these URLs and generates the standards and informational lists from them.
- **Review depends on Word's change tracking and document comparison** (the redline PDFs are produced with them).
- **Alternatives have been tried and abandoned:**
  - AsciiDoc rendered to PDF: diffs of plain text full of markup got in the way of review.
  - LibreOffice: its document comparison is less capable than Word's.
  - Some other standards organizations use AsciiDoc on GitHub, with comments and edits made as pull requests. That is not considered realistic for the PWG at this stage.
- **Open-source tension.** Word is a commercial product, which sits uneasily with the PWG's culture of open source and open standards. It nevertheless remains the working tool, and the 2021 draft Communications Policy [PWG-COMM-POLICY-DRAFT] says "no going back to AsciiDoc / Git / GitHub."

The renovated site therefore treats the Word-on-archive workflow as fixed (see R-14).

### Related PWG Infrastructure

The web site is one part of a wider set of PWG communications facilities, inventoried in the PWG Communications Policy [PWG-COMM-POLICY], with more detail in its 2021 draft [PWG-COMM-POLICY-DRAFT]:

- **Domains:** `pwg.org`, with the host names `www.pwg.org` and `ftp.pwg.org`.
- **Role email addresses:** `chair@`, `vice-chair@`, `secretary@`, `pr@` and `webmaster@pwg.org`. These forward mail only; there is no webmail, so officers cannot send mail *from* these addresses.
- **Mailman mailing lists:** the public, self-subscribed lists with public archives (`pwg@`, `pwg-announce@`), workgroup lists (`ipp@`, `ids@`, `ippeveselfcert@`, `3d-printing@`), and the private Steering Committee list (`sc@`, with no archives and no self-subscription). The policy notes the server's "interesting way" of hosting archives, and the 2021 draft also calls it "pretty old". Since then, list mail has had growing delivery problems related to SPF, DKIM and DMARC (see Problems, section 6).
- **GitHub (`istopwg` organization):** `website`, `ippeveselfcert`, `ippsample`, `ippregistry`, `pwg-books` and `pwg-semantic-model`.
- **Build automation:** Travis CI was listed in 2021. *(Its free service for open source, travis-ci.org, shut down in 2021, so current status should be checked.)*
- **Code signing certificates** for official software releases, such as the IPP Everywhere Self-Certification tools.
- **Teleconferencing** through Webex, not hosted by the PWG.

Since the policy was written, the PWG has also obtained an **ICDSoft managed hosting account**. The goal is to administer email addresses and other infrastructure through cPanel instead of over SSH on the Linode VM. ICDSoft is responsible for operating-system updates, PHP updates and similar maintenance. Moving from the current system to ICDSoft has run into a few gaps, and the move is not yet complete (see Problems, section 6, and Open Questions).

### Contributors and Maintenance

The Git history has about 1,000 commits, made almost entirely by two maintainers. The dynamo code last changed in 2025. Most recent activity is content updates: IPP draft listings and IPP Everywhere printer additions.

## Problems with the Current Site

*This list is preliminary. It will be revisited once the design requirements are written down, since some items may matter more or less depending on what the PWG decides it needs.*

### Context

The current architecture was designed and built roughly ten years ago (around 2014–2016). Many of its choices were reasonable then and some remain sound. In particular, **writing most pages as plain HTML is not a problem**: the PWG is comfortable editing HTML, and a content management system such as Drupal or WordPress is not wanted for most content. The problems below concern how that HTML is published and served, the age of the supporting software, and how the database-backed features are tied into the rest of the site.

### 1. Static Content and the Dynamic Application Are Tightly Coupled

- **Every HTML page runs through PHP.** Because Apache routes all `.html` requests through `dynamo/wrap.php`, even a simple informational page depends on the PHP application, its configuration and (for login state) its database connection. A fault in dynamo, or a PHP upgrade that breaks it, can take down the whole site, not just the dynamic features.
- **Site layout and navigation are in PHP code.** The header, menus and footer are hard-coded strings in `dynamo/phplib/site.php`. Adding a page to a menu, renaming a workgroup or changing the footer requires editing PHP instead of HTML.
- **One server, one security boundary.** The public informational pages, the login system, admin functions and document uploads share one host, one PHP runtime and one set of credentials. Any weakness in the dynamic code exposes the entire site.
- **The dynamic features are a small, separable part of the site.** Only a few functions really need server-side code and a database: accounts and roles, the document registry (and the standards and informational lists generated from it), news articles, issue tracking, and document uploads to the FTP server. Everything else is static. The current design does not reflect this split. (Whether those functions should move to a separate subdomain, such as a members or tools site, is a design question for a later section.)

### 2. Aging Software Platform

- **The PHP code does not run on currently supported PHP versions.** All eight data classes (`article`, `comment`, `document`, `issue`, `organization`, `printer`, `user`, `workgroup`) use PHP 4-style constructors, where the constructor is a method named after the class. PHP 8.0 no longer treats those methods as constructors, so `new document($id)` would quietly return an empty object, and pages such as the standards list and home page news would render wrong or empty without any error. The code also uses `each()`, which was removed in PHP 8.0, in the bulk-edit handlers of `documents.php`, `accounts.php`, `organizations.php` and `workgroups.php`. It also uses `${var}` string interpolation (39 occurrences in `site.php` alone), which is deprecated as of PHP 8.2. It depends on `short_open_tag = On` and on emulating long-removed PHP global variables. In practice this ties the site to PHP 7.x or earlier unless the code is updated. Production actually runs PHP 5.4.16 (see Overview), which stopped receiving security fixes in 2015.
- **Front-end libraries are years out of date.** Bootstrap 3.3.7 reached end of life in 2019. jQuery 3.2.1 predates fixes for known cross-site scripting vulnerabilities (fixed in jQuery 3.5.0). The footer still loads the retired `www.google.com/jsapi` loader. Unused copies of Bootstrap 3.2.0 remain in `dynamo/resources/`.
- **There is no dependency management.** Libraries are pinned by hand-written CDN URLs scattered across PHP and HTML files (the printers page loads its own, different jQuery version), so upgrades must be found and made by hand.
- **Legacy compatibility code is still in place,** such as the `X-UA-Compatible: IE=9` header and the old `PWGContentBody` page-wrapper format that `wrap.php` still parses.

### 3. Content Is Spread Across Several Systems

- **Web content is split between Git and the database.** Keeping documents on `ftp.pwg.org` is intentional and is not a problem: it gives long-lived, dependable URLs. The web site's own content, however, is divided between HTML pages in Git and news, document metadata, users and issues in the MariaDB database. (According to the README, the live copy of the static pages also sits in the archive's `pub/pwg/www` directory, which mixes web site files with the document archive.) No single place shows what the web site contains.
- **The publishing path is unclear.** The README says only `dynamo/` changes are deployed from Git and that other static content is served from the FTP server's `pub/pwg/www` directory. Recent content commits suggest otherwise. Contributors cannot easily tell whether a merged change is live.
- **There is no preview or staging.** A change is either live or not. Setting up a local copy means configuring Apache, PHP and a database by hand.
- **Some page lists are kept by hand.** Blog posts and meeting pages are individual HTML files with no generated index, archive, tags or RSS/Atom feed. Blog posting appears to have stopped in 2021. The printer list (about 2,000 entries) is edited by hand as one large JSON file.
- **Pages are inconsistent.** Some pages are fragments wrapped by `wrap.php`, while others (for example `printers/index.html`) are standalone documents with their own styles and scripts, so the look and behavior vary across the site.

### 4. Security and Operations

- **Weak deployment webhook.** `github.php` checks only GitHub's older SHA-1 signature header and compares signatures with an ordinary string comparison (not a constant-time one). It runs `git pull` as the web server user, which means the web server can write to its own code.
- **Unencrypted document uploads.** `document::upload()` copies files into the archive at `ftp://user:password@ftp.pwg.org/...` using the plain FTP protocol, so the archive's write credentials and the file contents cross the network unencrypted. The archive itself should stay; the concern is only how files are written to it. (Reading files is usually done over HTTP/HTTPS.)
- **Dated password storage.** Passwords are hashed with salted SHA-512 `crypt()`, not a modern password-hashing function such as bcrypt or Argon2 (PHP's `password_hash()`).
- **Fragile email delivery.** Account emails and notifications are sent with PHP's `mail()`, which depends on the host's local mail setup.
- **No automated checks.** There is no CI, no tests, no link checker and no HTML validation, so regressions and broken links are found only by visitors.
- **Few maintainers.** Nearly all commits come from two people, and the custom framework has no outside documentation or community.
- **Hand-managed single server.** The web site, FTP archive, email forwarding and mailing lists all run on one Linux VM that must be patched and administered by hand. The 2021 draft Communications Policy [PWG-COMM-POLICY-DRAFT] already named this as something to move away from: services should not require specialist skills, one volunteer's time, or manual VM management.

### 5. Functional Defects and Limitations

- **JPEG files are mislabeled.** In `wrap.php`, the JPEG test is written as `/\.(jpg,jpeg)$/` instead of `/\.(jpg|jpeg)$/`, so it never matches. (It is only reached if a JPEG request is routed to the wrapper.)
- **Search depends on Google.** Search uses an embedded Google Programmable Search box, which brings third-party tracking and ads, offers limited control over results, and does not index content behind login.
- **Broken links and dead third-party resources.** A W3C Link Checker run on the home page (November 2026, one document, not recursive) reported a 404, a failed connection and two redirects. All but one come from the shared template, so they affect **every page**:
  - `https://www.google.com/cse/style/look/default.css`, loaded in every page's `<head>` (`site_header()`), returns **404**.
  - The privacy-policy link `https://ieee-isto.org/privacy-policy/`, in both the About menu and the footer, is **dead**. IEEE-ISTO renamed itself ISTO and did not keep `ieee-isto.org` redirecting, and the domain no longer resolves in DNS. **To fix:** all five live references (`site.php` ×2, `pwg-cookie-notice.js`, `ipp/iana-registry.xsl` ×2) should become `https://isto.org/privacy-policy/`, which works.
  - About 20 historical meeting pages (2000–2011, under `chair/meeting-info/`) link to `www.ieee-isto.org/pwg/registration.*` and `intranet.ieee-isto.org`. Those registration pages don't exist on `isto.org` either (404), so changing the domain would not help. *(Decide whether to leave them as historical record, or remove the links but keep the text.)*
  - Several current pages and files still use the old name "IEEE-ISTO" in their text (`standards.html`, `namespaces.html`, `chair/participating.html`, `NOTICE`, `ipp/iana-registry.xsl`). Historical blog posts should keep the name they were published with. *(Updating the name in current pages, especially copyright and legal notices, should be checked with ISTO.)*
  - `https://www.google.com/jsapi`, loaded in every page's footer, **permanently redirects** to Google's unrelated Charts loader, and nothing in the footer uses it.
  - The home page links to `/3d` without a trailing slash, which causes a redirect to `/3d/` (minor; in `dynamo/index.php`).

  Other checks of the template and pages found:
  - The standalone `search.html` page uses the long-retired Google AJAX Search API (`google.load('search')`), loaded over plain `http://`, so it no longer works. The navbar's search dialog uses the current `cse.js` and does work.

  *(A recursive run, using the online checker's "Check linked documents recursively" option, which is limited to 150 documents, or `checklink --recursive` from the command line, is still needed to find broken links inside individual pages.)*
  - The placeholder `informational.html` still references files that don't exist (`dynamo/resources/jquery-1.11.1.min.js`, `presentations/...`). It is normally replaced by `informational.php`, but it is stale.
- **Settings are hard-coded.** Standards series names (5100–5110) are hard-coded in `standards.php`, and the AWStats link and sender addresses are hard-coded in scripts.
- **Mixed `http://` and `https://` links.** Some page content and configuration templates still use `http://` URLs and old domain names (for example `www.isto.org`).
- **No private or embargoed self-certification submissions.** Since 2020, every IPP Everywhere listing is a hand edit to a public JSON file in a public Git repository. Vendors cannot submit results privately, cannot set a release date, and cannot see or manage their own submissions. The listing volunteer has to do every addition by hand, and anything committed is public immediately, including in the repository's history. This is a frequent request from submitters.

### 6. Mailing Lists and Hosting Migration

These problems sit mostly outside the web site, but they affect hosting decisions for it.

- **Mailing list mail is increasingly rejected or marked as spam.** Over the last few years, more and more receiving mail systems enforce SPF, DKIM and DMARC. Mailing list servers have a built-in conflict with these checks:
  - The list re-sends each post from its own server, which is not listed in the original sender's SPF record.
  - Mailman typically changes the message (for example, adding a `[IPP]` subject prefix or a footer), which breaks the original sender's DKIM signature.
  - When the sender's domain publishes a strict DMARC policy (`p=quarantine` or `p=reject`, which is now common for large mail providers and corporate domains), receiving systems may reject or quarantine list mail. Repeated bounces can then get subscribers automatically unsubscribed.

  Fixes generally include rewriting the `From:` header for those senders, DKIM-signing outgoing list mail for `pwg.org`, and publishing correct SPF, DKIM and DMARC records for `pwg.org`. Older Mailman installations support these only partly. *(The current configuration of the PWG list server should be documented here.)*
- **Incomplete migration to managed hosting.** The ICDSoft account is meant to replace hand administration of the Linode VM, but the move has gaps, and until it is complete the PWG pays for and maintains two environments. *(The specific gaps should be listed here.)* For the web site specifically, the following are likely compatibility points:
  - dynamo requires PHP 7.x or earlier (production runs 5.4; see section 2), while a managed host keeps PHP up to date.
  - Page wrapping depends on Apache's `Action` directive (`mod_actions`).
  - Deployment relies on a webhook that runs `git pull` on the server.
  - Hosting the FTP archive, including anonymous `ftp://` access, at its existing URLs.

## Requirements

*This section is being drafted. Numbered requirements are tagged by source:*

- *[POLICY]: the 2021 draft of the PWG Communications Policy [PWG-COMM-POLICY-DRAFT]. The policy was approved on May 1, 2023 [PWG-COMM-POLICY], but the drafts' plans for future changes were removed from it so it could be finalized in time for PWG Process 4.0 [PWG-PROCESS]. The approved policy mainly records the role email addresses and mailing lists. Those plans were set aside, not rejected, and they record the Steering Committee's positions as of 2021. Requirements tagged [POLICY] should therefore be confirmed by the Steering Committee before they are treated as settled.*
- *[SC]: positions stated during this renovation effort.*

*The key words MUST, SHOULD and MAY are used as defined in [BCP14].*

### Functions

#### Static Content

There are many pages that provide effectively static page content. They don't need to live in a CMS. It is good that they are in 

### Scope

[PWG-COMM-POLICY] and its 2021 draft [PWG-COMM-POLICY-DRAFT] cover all PWG communications facilities. This renovation covers the **web site** and the web functions it hosts. The other facilities they list (role email, mailing lists, file sharing, teleconferencing, document editing) are listed under "Related Infrastructure Requirements" so the design can avoid conflicting with them and can share hosting decisions where that makes sense. They are not otherwise in scope.

### General

- **R-1** [POLICY] The web site MUST be the "first stop" for the PWG: the public entry point that gives access to PWG documents, specifications, current work and web applications.
- **R-2** [POLICY] Managing and maintaining the site MUST NOT require a computer science degree or other esoteric skills, depend on any one volunteer's time, or require someone to manage a Linux VM by hand.
- **R-9** [SC] The site SHOULD be deployable on the PWG's ICDSoft managed hosting account (administered through cPanel, with operating-system and PHP updates handled by ICDSoft), or on another service needing no more administration. Any server-side code MUST keep running across the host's routine PHP upgrades.

### Content and Authoring

- **R-3** [SC] Most site content MUST be maintainable by editing HTML files directly. A content management system such as Drupal or WordPress MUST NOT be required for general content.
- **R-4** [POLICY] Static content MUST be supported as the primary form of site content.
- **R-10** [SC] Contributors who are not comfortable with HTML MUST have a supported way to create and update pages, for example by writing in Markdown or a visual editor, or by converting from Word or LibreOffice.
- **R-11** [SC] Whatever authoring method is used, the stored page source MUST stay clean and editable by hand. Tools that produce bloated or tool-specific HTML SHOULD be avoided.
- **R-12** [SC] Contributors SHOULD be able to create and edit pages from a web browser, with changes reaching the live site without installing local tools or uploading files by hand.
- **R-13** [SC] Any authoring system MUST NOT add a server-side application, database or plugin ecosystem that the PWG has to secure and patch itself. Every change MUST be attributable to a person and reversible.

### Document Archive

- **R-5** [SC, POLICY] Existing `ftp.pwg.org` URLs MUST keep working. The site MUST link to documents in the archive and MUST NOT rely on moving or copying them.
- **R-6** [POLICY] Archive files MUST stay available over HTTP/HTTPS, and anonymous `ftp://` access SHOULD stay available for older documents that embed `ftp` URLs.

### Web Applications

- **R-7** [POLICY, SC] The site MUST provide a way for readers to report errata against published PWG documents, and a way for the PWG to track, respond to and resolve those reports (today, `dynamo/issues.php` with its database). The 2021 draft Communications Policy [PWG-COMM-POLICY-DRAFT] asks whether this could become a GitHub project; see Open Questions.
- **R-8** [POLICY] The site MUST provide a public, searchable listing of IPP Everywhere self-certified printers (today, a hand-edited `printers/printers.json` filtered in the browser).
- **R-15** [SC] The site MUST provide a way to register successful IPP Everywhere self-certification results:
  - **R-15.1** Only authorized submitters (for example, people whose organization is entitled to submit) can make submissions.
  - **R-15.2** A submission MUST be able to stay **private until a submitter-chosen release ("turn on") date**, then appear in the public listing automatically, without a manual step on that date.
  - **R-15.3** Before release, a submission's details MUST NOT be visible publicly, including in public source repositories, their commit history, or published data files.
  - **R-15.4** Submitters SHOULD be able to view and correct their own pending submissions.
  - **R-15.5** Submissions SHOULD include the self-certification result files and SHOULD be validated automatically, as the 2015–2020 system did.
  - **R-15.6** Newly released printers SHOULD continue to be announced (as `evenotify.sh` did for `pwg-announce`).
- **R-16** [SC] The site MUST keep a news and blog system:
  - **R-16.1** Authorized people (today, officers and admins) can post news items with a title, summary, date, optional workgroup and optional link.
  - **R-16.2** The site can host full blog articles (meeting summaries, press releases and similar).
  - **R-16.3** The home page shows recent news, and a news item can be **pinned until a given date** (today's display-until date), after which it drops into the normal news list automatically.
  - **R-16.4** All past news and blog posts stay browsable, and existing `blog/` URLs MUST keep working.
  - **R-16.5** The site SHOULD offer an RSS/Atom feed.
  - **R-16.6** Publishing a post SHOULD be a single step in a single place, without separate entries in separate systems. Sending the matching `pwg-announce` email from the same source MAY be supported.
  - **R-16.7** Posting MUST meet the authoring requirements R-10 through R-13.

### Related Infrastructure Requirements

These are recorded from the 2021 draft Communications Policy [PWG-COMM-POLICY-DRAFT] for context. They are outside this renovation's scope except where the site depends on them.

- Role email accounts for officers (Chair, Vice-Chair, Secretary, Webmaster and others), with a way to *send* from those addresses (for example webmail), not just forwarding.
- A mailing list server with lists for every workgroup and `pwg-announce`, self-subscription and open archives. It also needs a private `sc` list with no self-subscription and no public archives. [SC] List mail must be delivered reliably to subscribers whose mail systems enforce SPF, DKIM and DMARC.
- Private file sharing (candidates named: FTPS, SFTP, SharePoint, Google Drive, Nextcloud).
- Teleconferencing stays with an outside provider (currently Webex).
- Specifications continue to be edited in Microsoft Word, PowerPoint and PDF (possibly Google Docs). The policy states "no going back to AsciiDoc / Git / GitHub" for specification authoring. This does **not** conflict with R-3: specifications are not authored on the web site, and the web site's own pages can still be HTML kept in Git.

### Standards Development Support

- **R-14** [SC] The site MUST support the existing standards workflow: Word working drafts with clean and redline PDFs posted to each workgroup's `wd/` folder on the archive, and published standards in `/pub/pwg/standards/`. The renovation MUST NOT require a change in how specifications are authored or reviewed. (Document types, working-draft phases and approvals are defined in [PWG-DOC-MGMT-POLICY], whose examples use the same `wd/` folder URLs.)

### Open Questions

1. **Separate the web applications from the main site?** Should the database-backed functions (accounts, document registry, errata, news, uploads) move to a separate subdomain or service, so the main site can be purely static? [SC]
2. **Errata as a GitHub project?** Could errata reporting use GitHub Issues instead of a custom system? [POLICY] What would that mean for reporters who don't have GitHub accounts?
3. **IPP Everywhere submission system.** R-15 (private submissions with a release date) rules out managing pending submissions in a public Git repository. What should hold them? Candidates:
   - **(a)** A small server-side application (for example on an ICDSoft web-applications subdomain) that stores pending submissions privately and publishes released entries.
   - **(b)** A *private* GitHub repository or private form intake, plus a scheduled GitHub Action that copies entries past their release date into the public `printers.json` and rebuilds the site.
   - **(c)** Another hosted form-and-database service.

   Option (b) keeps the public site static but makes the PWG depend on GitHub for confidentiality. Option (a) gives submitters accounts and self-service, but keeps a server-side application to maintain (R-13).

   **Current leaning: (a)**, as a `certification.pwg.org` subdomain that serves the public registry JSON to the static site (see Design Options). Still to decide: who may submit (any member organization, or named people), whether a PWG reviewer approves submissions before they can be released, and whether to adapt the old dynamo code or start fresh.
4. **How private must the `sc` list be?** The 2021 draft Communications Policy [PWG-COMM-POLICY-DRAFT] itself asks, "Isn't everything public?"
5. **Accounts.** Which remaining functions still need PWG-managed user accounts and roles once the questions above are decided?
6. **ICDSoft migration gaps.** What exactly blocked or complicated the move to ICDSoft? Which services (web site, web applications, FTP archive, role email, mailing lists) can ICDSoft host, and which will need another home?
7. **Mailing list platform.** Can the lists be fixed in place (DMARC mitigation, DKIM signing, DNS records), should they move to a newer Mailman on ICDSoft if available, or should they move to an outside list service? Either way, existing list addresses and archive URLs should be kept.
8. **Authoring workflow.** Who will edit site content, how comfortable is each of them with HTML, Markdown and Git, and which workflow (see "Authoring Workflow" under Design Options) serves the least technical regular contributor without breaking R-13?
9. **Document archive size and hosting.** How large is the `ftp.pwg.org` archive, and does anyone still depend on `ftp://` protocol access (as opposed to `https://` URLs with the same paths)? Does ICDSoft offer enough storage, and anonymous FTP if needed?
10. **Shared sign-in.** How many PWG-run applications will need sign-in, and who uses them? Is requiring a GitHub, Google or Microsoft account acceptable for all of those users, or must the PWG offer its own accounts? (See "Shared Sign-In Across Subdomains" under Design Options.)
11. **IPP demonstration services.** Is there interest (and an owner and budget) for hosting public IPP demonstration services? Which services would be most useful to implementers? (See "Future Possibility: IPP Demonstration Services" under Proposed Architecture.)
12. **Repository layout.** Should the served site move into a `www/` folder so that project files are never published, and should future subdomain sites (`certification.pwg.org`, `auth.pwg.org`) live in the same repository or in separate ones? Who should review changes to each site? (See "Repository Layout: Serve Only a `www/` Folder" under Design Options.)

## Proposed Architecture

*Draft for discussion. The aim is the **fewest moving parts** that meet the requirements. Everything else is deferred until a real need appears. The "Design Options" section below contains the supporting analysis and the alternatives that were considered.*

### Guiding Patterns

1. **Static first.** If content can be a file, it is a file: pages, news, blog posts, meeting pages, and the standards and informational lists (as data files). A static site generator builds the site, and GitHub Actions deploys it automatically whenever the main branch changes.
2. **Git is the database for public content.** History, attribution, review (pull requests) and rollback come built in, which meets R-13 with no server, database or plugins.
3. **One scheduled daily rebuild handles every time-based need,** such as news pinned until a date. No per-feature scheduling logic is needed.
4. **Run a server application only where confidentiality requires it.** Today that is IPP Everywhere certification (R-15.3). The application *publishes data* (a public JSON registry) that the static site reads. The two are not integrated in any other way.
5. **Borrow; don't build.**
   - Sign-in uses existing identity providers.
   - Errata use GitHub Issues with a structured issue form, plus an email address for reporters who don't have a GitHub account.
   - Email and mailing lists use a host or service, not software the PWG runs itself.
6. **Keep URLs.** Existing `www.pwg.org`, `blog/` and `ftp.pwg.org/pub/pwg/...` URLs keep working (R-5, R-16.4).

### Components

| # | Component | Holds | Runs on |
|---|-----------|-------|---------|
| 1 | **`www.pwg.org`**: static site built from GitHub | All pages, news and blog, standards and informational lists (data files), navigation, IPP Everywhere listing page | GitHub Pages or ICDSoft (static files only) |
| 2 | **`certification.pwg.org`**: small application | Private and pending self-certification submissions, submitter sign-in, public `registry.json` | ICDSoft (managed PHP + database) |
| 3 | **`ftp.pwg.org`**: document archive | Specifications, drafts, minutes, charters, at their existing paths | ICDSoft or equivalent storage |
| 4 | **Email and mailing lists** | Role addresses, workgroup lists, `pwg-announce`, archives | ICDSoft or a list service (Open Question 7) |

With these in place, the Linode VM and nearly all of dynamo can be retired.

### What Each Requirement Maps To

| Need | Handled by |
|------|------------|
| Static pages, edited as HTML or Markdown (R-3, R-10–R-12) | Component 1; GitHub web editor first, a Git-based CMS only if authors need it |
| News and blog, with pinning and a feed (R-16) | Component 1, as generator posts, plus the daily rebuild |
| Standards and informational lists (R-14) | Component 1, from a data file that lists document URLs on the archive |
| Errata (R-7) | GitHub Issues with an issue form, plus an errata email address |
| IPP Everywhere registration with release dates (R-15) | Component 2, read by the listing page on Component 1 |
| Document archive with stable URLs (R-5, R-6) | Component 3 |
| Low maintenance (R-2, R-9, R-13) | Only Component 2 has server-side code, and it runs on managed hosting |

### Deliberately Deferred

- **`auth.pwg.org` single sign-on.** It isn't needed while only one application requires sign-in. Revisit when there is a second one, or if the IPP demonstration services go ahead, since they would use OAuth and benefit from the same authorization server.
- **A Git-based CMS.** Start with GitHub's web editor and add a CMS only if contributors find that too hard (Open Question 8).
- **Self-hosted services of any kind** (identity, mailing lists, CMS).

### Trade-offs to Confirm

- **Document uploads change.** dynamo currently uploads files to the archive *and* records them in the database in one step. In this design, editors upload files to the archive separately (SFTP or the cPanel file manager), and someone adds the entry to the standards data file in Git. Is that acceptable to document editors, or does document registration need to be part of Component 2?
- **Errata reporters without GitHub accounts** report by email, and a PWG volunteer files the issue for them.
- **User accounts disappear for everything except certification.** Member-only functions that exist today (for example "Request Access" roles) need to be checked to confirm nothing else depends on them (Open Question 5).

### Future Possibility: IPP Demonstration Services

*Outside the scope of the web site renovation; recorded so the architecture doesn't preclude it.*

The PWG could host live, public IPP services as demonstrations and interoperability targets for developers. Examples:
- a public IPP Everywhere test printer;
- a shared-infrastructure (cloud) print service using the IPP Shared Infrastructure Extensions (INFRA, PWG 5100.18), with a proxy;
- an IPP System Service (PWG 5100.22) endpoint.

Much of the software already exists in the PWG's `ippsample` project (`ippserver`, `ippproxy`, `ipptool`).

Considerations:
- **Hosting.** IPP services are long-running network servers (normally on port 631), which shared web hosting generally cannot run. They need a container platform or a VM. A container platform that deploys from a GitHub repository keeps this closest to R-2, since it avoids hand-managing a server.
- **Abuse resistance.** A public print endpoint accepts uploads from anyone. It needs authentication or rate limiting, job size limits, automatic deletion of submitted documents, and periodic reset to a clean state.
- **OAuth.** The natural authentication method is OAuth 2.0: IPP clients get an access token from a PWG authorization server before submitting jobs. This controls access with a standard mechanism, and it also gives implementers a working end-to-end demonstration of IPP with OAuth. The authorization server could be the same one that provides shared sign-in for PWG web applications (see "Shared Sign-In Across Subdomains").
- **Isolation.** Run it on its own subdomain and infrastructure, separate from Components 1–4, so a problem with the demo cannot affect the web site, the archive or certification.
- **Ownership and lifetime.** It needs a named maintaining group (likely the IPP workgroup), a budget, and an agreed way to retire it if it is no longer maintained.

## Design Options

*Approaches under consideration for the restructured site (Phase 3 and later). Not yet decided.*

### Static Site Generator (for Example Jekyll)

A static site generator builds the whole site into plain HTML files ahead of time, so the web server only serves files and runs no PHP. Jekyll is the generator built into GitHub Pages; Hugo and Eleventy are common alternatives.

- **Close fit with today's pages.** `wrap.php` already treats each page as a title, an optional subtitle and an HTML body wrapped in a shared header and footer. A Jekyll page has the same structure: a short front-matter block (title, subtitle) followed by the HTML body, wrapped by a shared layout. Existing pages could be converted largely by script, and authors would keep editing HTML (R-3).
- **Navigation becomes data.** Menus would move from PHP strings in `site.php` into a data file and a layout template, so changing navigation no longer means editing code.
- **Index pages can be generated** for the blog, meetings and similar lists, along with an RSS/Atom feed.
- **News and blog map naturally onto a generator's posts** (R-16). Each news item or blog article becomes one file whose front matter holds the title, date, summary, workgroup, optional external link and optional `pinned_until` date. The generator builds the home-page news list, pinned alerts, archive pages and feed from those files, so the database and the separate dynamo news step go away. Because a static site only changes when it is rebuilt, honoring pinned-until dates (and R-15.2 release dates) needs a **scheduled daily rebuild** (for example a GitHub Actions scheduled job). Existing `article` records can be exported once into post files.
- **Fits R-2.** There is no PHP runtime or database to maintain for the content pages.
- **Limit: no server-side code.** Accounts, document uploads and any other database-backed functions cannot run on a static site. Some data could be kept as data files in Git and rendered at build time (for example the standards and informational lists, from a YAML or JSON file instead of the `document` table). Others would need a separate home or a replacement (see Open Questions 1, 2 and 5).

### Authoring Workflow

This is the classic tension between **usability and simplicity**. The simplest site is static HTML, and contributors who can edit HTML are well served by it. Not every PWG contributor is comfortable with HTML, though, and each step toward easier authoring tends to add either messy page source or software to maintain. A traditional CMS (WordPress, Drupal) solves usability but brings a server-side application, a database and plugins that must be patched constantly, which is exactly the security and maintenance burden the PWG wants to avoid (R-2, R-13).

The options, from simplest to most usable:

| Approach | How authors work | Advantages | Drawbacks |
|----------|------------------|------------|-----------|
| **Hand-edited HTML** (today) | Edit HTML files, commit to Git | Simplest; full control; no tools | Needs HTML and Git skills |
| **Visual HTML editors** (for example SeaMonkey Composer) | Edit pages visually, then commit or upload the file | No HTML knowledge needed | Often produces bloated, tool-specific HTML that becomes hard to edit by hand (conflicts with R-11); still needs a commit or upload |
| **Word or LibreOffice, converted** | Write in a word processor, convert to HTML or Markdown (for example with Pandoc) | Familiar tools; the PWG already writes specifications in Word | Conversion output needs cleanup; a two-step workflow; still needs a commit or upload |
| **Markdown with a static site generator** (for example Jekyll) | Write Markdown (plain text with light formatting; raw HTML still allowed where needed) | Easy to learn; clean, readable source | Changes must be committed and pushed, and the site rebuilt |
| **Editing in the browser on GitHub** | Edit a Markdown or HTML file on github.com and commit it; GitHub Actions rebuilds and deploys automatically | No local tools; history and review through pull requests; nothing extra to run | Needs a GitHub account and basic familiarity with GitHub; no visual editor |
| **Git-based CMS** (for example Decap CMS, Pages CMS, TinaCMS) | Log in to a web editor with forms and a visual or Markdown editor; each save becomes a Git commit, followed by an automatic rebuild | CMS-like usability; the site stays static; no PWG-run database or server-side application; full Git history | Another tool to choose and configure; usually needs GitHub sign-in (and for some products a small authentication service or a hosted third-party service) |
| **Traditional CMS** (WordPress, Drupal) | Edit in a web interface that publishes immediately | Most familiar editing experience | Server-side application, database and plugins to secure and patch; conflicts with R-2 and R-13 |

**"Doesn't that require a push?"** With a static site generator, every change does need a commit. But the commit doesn't have to come from a local Git checkout and a manual upload. If GitHub Actions builds and deploys the site automatically on every change to the main branch, then an edit made in GitHub's web editor or in a Git-based CMS is live a minute or two later. Pull requests can also be used for review before a change goes live, where that is wanted.

**Possible direction (not yet decided):** keep static pages as the base, written in HTML or Markdown as each author prefers. Rebuild and deploy automatically from GitHub. Let contributors who don't want to deal with Git edit through GitHub's web editor or a Git-based CMS. This meets R-10 through R-13 without running a CMS server.

### Hosting Static Output: GitHub Pages or ICDSoft

Choosing a generator and choosing a host are separate decisions. The generated site can be built by GitHub Actions on every merge and published either way:

- **GitHub Pages:** free, needs no server administration, and supports a custom domain (`www.pwg.org`) with HTTPS. Limits: no server-side redirects or custom headers (the `/issues` redirects in `.htaccess` would need redirect pages), a soft 1 GB site-size limit, and GitHub's own plugin restrictions unless the site is built with Actions.
- **ICDSoft:** the generated files are uploaded to the managed host (over SFTP or FTPS). `.htaccess` redirects keep working, and a separate web-applications subdomain could live on the same account.

Either way, `ftp.pwg.org` remains a separate host and keeps its existing URLs (R-5).

### IPP Everywhere Certification Subdomain (`certification.pwg.org`)

*Leading candidate for R-15 and Open Question 3.*

A small, separate web application at `certification.pwg.org` owns self-certification submissions. The main site stays static and only *reads* the public registry from it.

```
 Submitters (logged in)                       Public visitors
        │                                            │
        ▼                                            ▼
 certification.pwg.org                         www.pwg.org/printers/
  • login (authorized submitters)               (static page; filtering
  • submit results, set release date             in the browser, as today)
  • view/edit own pending submissions                │
  • validation of result files                       │  fetches
  • private store of pending entries                 │
  • GET /registry.json  ◄────────────────────────────┘
      (released entries only)
```

- **Release dates are enforced on the server when data is requested.** `registry.json` includes only entries whose release date has passed, so a printer appears the moment its date arrives, with no rebuild or manual step (R-15.2). Pending entries never leave the certification server and never touch a public repository (R-15.3).
- **Small change on the public side.** `printers/index.html` already fetches `printers.json` and filters it in the browser; it would fetch `https://certification.pwg.org/registry.json` instead. The certification server must send a CORS header (`Access-Control-Allow-Origin: https://www.pwg.org`) so the browser allows the cross-site fetch.
- **The JSON format can stay as today's** (`family`, `model`, `url`, `color`, `duplex`, finishings fields, `ipps`, `version`, `date` and so on). Then the existing page and any outside consumers of `printers.json` keep working. For compatibility, `www.pwg.org/printers/printers.json` can be kept as a periodically refreshed copy or a redirect.
- **Resilience.** If the certification site is down, the public list should not go blank. A scheduled job (for example daily, in GitHub Actions) can save a snapshot of the released registry to the main site as a fallback. The same snapshot lets a static site generator render the list into HTML, which search engines index better.
- **Migration.** The roughly 2,000 entries in today's `printers.json` are imported once as already-released entries.
- **Announcements.** The certification application can send the weekly `pwg-announce` summary of newly released printers (R-15.6), as `evenotify.sh` did.
- **Cost.** This is a server-side application with logins and a database. It is the one place the design accepts that burden (R-13), because confidentiality (R-15.3) cannot be met by a static site. It should run on managed hosting (ICDSoft, R-9), be kept small, and could be written from scratch or adapted from the 2015–2020 dynamo submission code (`evesubmit.php`, `printer_validate_plist()`) after the Phase 1 PHP 8 work.
- **Pattern for other applications.** If other server-side functions survive (Open Questions 1 and 5), the same pattern applies: a static `www.pwg.org`, plus small purpose-specific subdomains that publish data the static site reads.

### Shared Sign-In Across Subdomains (`auth.pwg.org`)

If more than one application ends up on its own subdomain (`certification.pwg.org` and possibly errata, document management or others), users should have **one PWG sign-in** across them (single sign-on). Two concerns should be kept separate:

- **Authentication** (who the user is) is handled once, centrally.
- **Authorization** (what the user may do: member, editor, officer, admin, certification submitter) is decided by each application from a PWG-maintained role list.

Approaches, from simplest to most capable:

| Approach | How it works | Advantages | Drawbacks |
|----------|--------------|------------|-----------|
| **Shared domain cookie** | Extend today's signed `PWGSID` cookie to `.pwg.org`; all applications share the user database and signing secret | Simplest; close to how dynamo already works | Every application holds the master secret, so a weakness in one exposes all; the cookie is sent to every `*.pwg.org` host; the PWG still stores passwords and sends reset emails |
| **Self-hosted OpenID Connect provider at `auth.pwg.org`** (for example Keycloak, Authentik, Zitadel) | Applications redirect to `auth.pwg.org` and receive a verified identity using a standard protocol | Standard, secure, works for any number of applications; supports multi-factor sign-in | Substantial server software, probably not runnable on shared hosting; a significant maintenance and security burden (conflicts with R-2 and R-13) |
| **Hosted OpenID Connect provider** behind `auth.pwg.org` | Same protocol, run by a vendor | Standard protocol without the server to maintain | Vendor dependency; possible cost; data held by a third party |
| **Sign in with existing accounts** (GitHub, Google, Microsoft) | Each application, or a small shared sign-in page, accepts these identity providers; the PWG keeps only a role list mapping identities to roles | The PWG stores no passwords and sends no password-reset email, which also avoids the email delivery problems in Problems, section 6; least to maintain | Every user needs one of those accounts (corporate users usually have Microsoft or Google accounts through work) |

**Possible direction:** size this to the number of applications that survive. With one or two (for example certification and errata), use sign-in with existing accounts plus a PWG-maintained role list. If the number grows, put a hosted OpenID Connect service behind `auth.pwg.org`; applications that already use OpenID Connect sign-in would need little change. A self-written authentication server is not recommended.

**Link to IPP demonstration services:** OpenID Connect is built on OAuth 2.0, so an authorization server at `auth.pwg.org` could issue tokens both to people signing in to PWG web applications and to IPP clients using PWG demonstration services (see "Future Possibility: IPP Demonstration Services"). If those services go ahead, that favors a PWG-controlled (probably hosted) OpenID Connect / OAuth server over relying only on GitHub, Google or Microsoft sign-in, since IPP services cannot easily accept tokens issued by those providers.

**Scope note:** shared sign-in covers PWG-built applications only. Third-party services the PWG uses (mailing lists, cPanel, GitHub itself) keep their own accounts unless they support the chosen protocol.

### Retiring the Linode VM: Where Each Service Goes

One proposal is to host everything on GitHub and shut down the Linode VM. GitHub can host the web site and some of what dynamo does today, but it cannot host email or mailing lists, and it is a poor fit for a large file archive. The Linode can be shut down only once **every** service it runs has a new home:

| Service | Today (Linode) | Can GitHub host it? | Candidate homes |
|---------|----------------|---------------------|-----------------|
| Static web site (`www.pwg.org`) | Apache + `wrap.php` | **Yes**: GitHub Pages, built by GitHub Actions | GitHub Pages or ICDSoft |
| Standards, informational and printer lists | Database (dynamo) and `printers.json` | **Yes, if converted** to data files in Git rendered at build time | GitHub Pages or ICDSoft |
| IPP Everywhere *pending* (embargoed) submissions | None since 2020 | **Only with a private repository** and a scheduled publish job; never in the public repository (R-15.3) | `certification.pwg.org` application on ICDSoft (leading candidate), or a private GitHub repository + Actions (Open Question 3) |
| News and blog | `article` table (dynamo) + `blog/` HTML | **Yes**, as generator posts with a scheduled daily rebuild for pinned items | GitHub Pages or ICDSoft |
| Errata / issue tracking | dynamo + database | **Possibly**, as GitHub Issues (Open Question 2) | GitHub Issues, or a small application on ICDSoft |
| Accounts, document uploads, admin pages | dynamo + database | **No** (needs server-side code) | Retire, or move to a web-applications subdomain on ICDSoft (Open Questions 1 and 5) |
| Document archive (`ftp.pwg.org`) | FTP + HTTP/HTTPS | **Poor fit** (see below) | ICDSoft, or object storage with a custom domain, keeping the `/pub/pwg/...` paths |
| Role email addresses | Forwarding on the VM | **No** | ICDSoft (cPanel), or a hosted email provider |
| Mailing lists | Mailman | **No** | Mailman on ICDSoft (if offered), a hosted Mailman service, or a list service such as Groups.io (Open Question 7) |
| Mailing list archives | Mailman (pipermail) | **Yes, for old archives**: Mailman 2's pipermail archives are already static HTML and could be served unchanged at their current URLs | GitHub Pages or ICDSoft for the frozen archives; the new list service for new posts |
| Usage statistics | AWStats on server logs | **No**: GitHub Pages gives no access to server logs | Analytics on the new host, a privacy-friendly analytics service, or none |
| DNS for `pwg.org` | *(to be confirmed)* | **No** | Registrar's DNS, ICDSoft or a DNS provider; it must carry the SPF, DKIM and DMARC records |
| Build automation | Travis CI (2021) | **Yes**: GitHub Actions | GitHub Actions |

**Why the archive is a poor fit for GitHub:**
- GitHub limits repository and Pages site sizes (Pages sites are capped at about 1 GB, individual files at 100 MB, and large files stored with Git LFS are not served by Pages). Twenty years of documents may exceed this. *(The archive's actual size should be measured.)*
- Git keeps every version of every file forever, which suits source code but not a document archive.
- Only `https://` URLs could be kept; `ftp://` URLs would stop working. That loss is smaller than it sounds, because current mainstream browsers dropped `ftp://` support around 2021. In practice the `https://ftp.pwg.org/pub/pwg/...` form is what most readers can still use, and that path layout can be kept on any new host.

**Likely end state:** GitHub for the web site, its source and its build, plus the PWG's ICDSoft account, which is already intended for email administration, for role email, the document archive and any remaining server-side functions. Mailing lists go on ICDSoft or a dedicated list service, depending on which handles DMARC better. The Linode VM is shut down after the last service moves.

### Repository Layout: Serve Only a `www/` Folder

Today the root of the repository is also the root of the web site, so everything in a checkout sits inside the served folder. That includes `README.md`, `CONTRIBUTING.md`, `NOTICE`, this design document and the `.git/` directory. The proposal is to move everything that is served into a `www/` folder and keep project files at the top level:

```
README.md, LICENSE, NOTICE, CONTRIBUTING.md
docs/           this design document, the test checklist (1.1), development setup notes
tools/          test and maintenance scripts that are not part of any site
www/            everything served at www.pwg.org, unchanged inside (dynamo/ included)
certification/  future certification.pwg.org application (see above)
auth/           future auth.pwg.org, if one is needed (see above)
```

Nothing changes *inside* `www/`, so every URL stays the same.

**What it fixes or enables:**

- **Project files are never served, on any host.** *Observed in November 2026:* a development system with its document root set to the checkout served the whole `.git/` directory, which means the full repository and its history, plus the Markdown files. Production returns 404 for these, but the rules that do it are not recorded in the repository. Phase 1 added `RedirectMatch` rules to `pwg-apache.conf` as a stopgap. With a `www/` folder the protection is built into the layout instead of depending on server rules, which matters on managed hosting where only `.htaccess` may be available (R-9).
- **A home for project material.** The design document, the test checklist and scripts, CI configuration and an optional Docker Compose file can live in the repository without being published.
- **Room for other sites.** `certification.pwg.org` and `auth.pwg.org` (if they are built) can each have a folder next to `www/` and share templates, styles and test tools with the main site. One pull request can then change a site and an application together.
- **A step toward a static site generator.** A generator keeps its source and its output in separate folders anyway. `www/` could later become the generator's source or the place its output is written.
- **A later step:** `dynamo/phplib/` and `dynamo/config/` could move out of `www/`, so that only the page scripts are inside the served folder. Their `include` paths would then need to change, so this is best done separately.

**What has to change:**

- **Web server configuration.** On development and staging systems, `DocumentRoot` becomes `checkout/www`. Production is set up differently: static content is uploaded by FTP to the archive's `pub/pwg/www` directory, and `dynamo/` is aliased in from the checkout (see Deployment and Hosting). There, uploads would come from `www/`, which is simpler than today because nothing outside `www/` should ever be uploaded. The `dynamo/` alias would have to point at `www/dynamo/`, which needs someone with administrative access to the server, since contributors have no SSH access. *(Production's exact setup should be confirmed with the infrastructure administrators before the move.)*
- **Automating uploads (optional).** A GitHub Actions job could upload `www/` to the live site on every merge, so the live pages can no longer drift from the repository. It must add and update files only, never delete them, because of the legacy content that is not in Git. It should wait until FTP encryption works, so the FTP password is not sent unencrypted from GitHub's servers.
- **The webhook.** `github.php` still runs `git pull` at the top of the checkout. `$SITE_DOCROOT` in `site.cfg` would keep meaning the checkout, not the web root, so it should be renamed (for example to `$SITE_CHECKOUT`) to avoid confusion.
- **Documentation and samples:** `pwg-apache.conf`, the setup instructions in `README.md` and the paths in development scripts.
- **Work in progress.** Git detects renamed files when merging, but branches started before the move will need a rebase or merge, and regular editors (for example of the `ipp/` pages) should be told in advance. Make the move as **one commit that only renames files**, with no content changes. That keeps Git's rename detection reliable and keeps `git log --follow` and `git blame` working across the move.

**One repository or several:**

| Approach | Advantages | Drawbacks |
|----------|------------|-----------|
| **One repository for all PWG sites** | Shared templates, styles and test tools; one place to look; suits a small volunteer team | Everyone with write access can change every site, unless GitHub's `CODEOWNERS` file and branch protection require the right reviewers for each folder. Each site needs its own deploy job, triggered by changes to its folder. Clones get bigger (already about 100 MB). |
| **A separate repository for each site** | Separate permissions and release schedules; smaller repositories | Shared templates must be copied or packaged; a change that spans sites spans several pull requests |

Either way, the certification application's *code* can be public. Pending submissions are *data* and never go in any repository (R-15.3).

**Timing:** not part of Phase 1, so the PHP 8 changes can be reviewed on their own. It fits best with the move to ICDSoft (Phase 2), because production's web root and deployment have to be set up again then anyway, so production's layout changes only once. It could be done sooner if the Linode setup is easy to change.

**Possible direction (not yet decided):** adopt the `www/` layout in its own pull request, as a rename-only commit, just before or during Phase 2. Start with one repository, using `CODEOWNERS` so each site folder has its own reviewers, and reconsider separate repositories if a site ends up with different maintainers. See Open Question 12.

## Upgrade Plan

The upgrade is staged so the site keeps working at every step, and so the work that is needed whatever the redesign decides can start before the open questions above are settled.

### Phase 1: Move the Existing Site to PHP 8

**Goal:** the current site, unchanged in structure and appearance, runs on a currently supported PHP 8 release (8.3 or later; 8.2 stops receiving security fixes at the end of 2026). This removes the main obstacle to moving to the ICDSoft managed host (R-9), where PHP is upgraded on the host's schedule, not the PWG's.

**Why first:** these changes are small, mechanical and low-risk; they don't depend on any open design decision; and every later phase benefits from a site that runs on a supported platform. Even if the database-backed functions are later redesigned or replaced, they must keep working in the meantime.

#### 1.1 Set Up a Test Environment

- **Development:** a local Linux system loaded from `dynamo/config/pwg.sql` and `test.sql`. The development machine is Ubuntu 26.04.1 LTS (x86_64), whose standard packages provide **PHP 8.5**, Apache 2.4.66 (including `mod_actions`) and MariaDB 11.8. PHP 8.5 is newer than what the managed host is likely to run, so code is developed and tested on 8.5, statically checked for 8.2 and later (PHPCompatibility `testVersion 8.2-`), and checked on the host's actual version on the staging site. A scan found none of PHP 8.5's new deprecations in the code (non-canonical casts such as `(integer)`, the backtick operator). A local mail catcher (such as Mailpit) captures the site's outgoing email for inspection. Optionally, a Docker Compose file in the repository would let any contributor run the same setup with one command.
- **Shared testing (staging):** a copy on a test subdomain of the ICDSoft account, protected by a password and excluded from search engines. Other reviewers can test there without any setup, and it shows problems specific to the host (PHP settings, `.htaccess` and `Action` support, outgoing mail) before production is touched.
- **Development setup notes** (from setting up the Ubuntu 26.04 system):
  - Ubuntu home folders are private, so run `chmod o+x` on the home folder and project folder for Apache to read the site.
  - Apache must be fully **restarted** (not reloaded) after adding the PHP settings file in `/etc/php/8.5/apache2/conf.d/`.
  - The repository has no root `index.html`. Production has a placeholder one, and `wrap.php` rejects requests for files that don't exist, so the development virtual host uses `DirectoryIndex index.html /dynamo/index.php`. (A bare `/dynamo/index.php` would also replace the index pages of folders such as `/ipp/` and `/printers/` with the home page.)
  - Ubuntu runs Apache with `MemoryDenyWriteExecute=yes`, so PHP logs a "PCRE JIT" warning on every page. Add `pcre.jit = 0` to the PHP settings file.
  - `loadtest.sh` ends with a harmless error about the "submission directory" (a leftover of the retired self-certification system; `SUBMISSION_DIR` is no longer in `site.cfg.tmpl`).
- **Test data:** use the test data or a sanitized copy of production. Real user records (email addresses, password hashes) should not be copied to the staging site.
- Write a short manual test checklist that covers each dynamo page and role: anonymous visitor, member, editor, officer and admin. It should cover the home page, standards and informational lists, login, password reset, account creation, document create/edit/upload, article management, issue tracking, and the organization, workgroup and user admin pages.

#### 1.2 Fix PHP 8 Breaking Changes

These are required: without them the site fails, or quietly misbehaves, on PHP 8.

| Change | Where | Fix |
|--------|-------|-----|
| PHP 4-style constructors are no longer constructors (silent failure) | 8 classes in `dynamo/phplib/db-*.php` | Rename each `classname()` constructor to `__construct()` |
| `each()` removed | Bulk-edit handlers in `documents.php`, `accounts.php`, `organizations.php`, `workgroups.php` | Replace `while (list($key, $val) = each($_POST))` with `foreach ($_POST as $key => $val)` |
| Short open tags (`<?`, `<?print`) depend on `short_open_tag = On`, which a managed host may not allow | `index.php` (10), `phplib/auth.php` (1) | Change to `<?php` or `<?=`, so the `short_open_tag` setting is no longer needed. *Observed on the development system:* with the setting off, every dynamo page prints the raw source of `auth.php` to the visitor, and all of its functions are undefined. (`site.cfg` uses `<?php`, so credentials are not exposed.) |
| Page options depend on `register_argc_argv = On`, which is off in Ubuntu's PHP settings and may be off on a managed host | Every dynamo page reads its URL options (`?U5`, `?L+I0`, `?B+I0+Q`, ...) from `$_SERVER["argv"]`, which PHP only fills in for web requests when this setting is on. *Observed on the development system:* with it off, all options are silently ignored. Edit links show the list page instead of the form, and batch updates do nothing, with no warning logged. | Build `$argc`/`$argv` in `phplib/site.php` from `QUERY_STRING` (split on `+`, as PHP does) when `$_SERVER["argv"]` is missing, so the setting is no longer needed |
| `dynamo/config/.htaccess` uses Apache 2.2 syntax (`Deny from all`), which `AllowOverride FileInfo Indexes` does not permit | Every request under `dynamo/config/` returns **500** on both production and the development system. This fails closed, so the files are protected, but only by accident. | Replace with `Require all denied` (allowing `AuthConfig` overrides), or deny the directory in the server configuration and remove the `.htaccess` |
| PDO's default error mode changed from silent to throwing exceptions (PHP 8.0) | `phplib/db.php` creates the connection without setting an error mode; callers check return values, not exceptions | To keep today's behavior, set `PDO::ATTR_ERRMODE` to `PDO::ERRMODE_SILENT` explicitly. Otherwise a database error that used to fail quietly becomes an uncaught exception and a blank or error page. Later, switch to exceptions with a logging handler. |
| Comparing a number with a non-numeric string changed meaning (PHP 8.0): `0 == ""` is now false | `phplib/site.php` `$LOGIN_ID != ""` (an integer) is now always true, so `Cache-Control: no-cache` is sent to anonymous visitors too (harmless but unintended). The other `!= ""` checks in the code compare strings and are unaffected. | Compare like with like (`$LOGIN_ID != 0`). Review loose `==` / `!=` comparisons between IDs and strings during testing. |
| `count()` / `sizeof()` on a non-array throws a `TypeError` (PHP 8.0) instead of a warning | About 25 call sites; nearly all are passed arrays. The plist validation in `db-printer.php` could be passed a missing `Tests` key. | Make sure values are arrays (for example `$x ?? []`) where the input may be missing |

#### 1.3 Clear Deprecations

These still work on PHP 8.3 and 8.4 but log warnings and are expected to break in a future release.

- Change `"${var}"` string interpolation to `"{$var}"` (61 occurrences, 39 of them in `phplib/site.php`).
- Harden form input handling. About 44 places pass `$_POST` or `$_GET` values straight to string functions (`trim()`, `strlen()` and others). On PHP 8 a crafted request that sends an array (for example `password[]=x`) causes a `TypeError` and an error page, where PHP 7 only warned. Check `is_string()` or cast before use.
- Optional cleanup (not required for PHP 8): drop the redundant `global $_SERVER, $_POST, ...` declarations (superglobals are always in scope) and the `$$var` emulation of old register-globals variables in `site.php`.
- Run every page from the test checklist with `error_reporting = E_ALL` and fix what gets logged. Likely candidates are passing `null` to built-in string functions (deprecated in 8.1) and undeclared ("dynamic") object properties (deprecated in 8.2).

#### Tooling for 1.2 and 1.3

Most of these changes are standard migrations with mature, free tools. Run the tools on the development system (they need PHP installed), review each diff, and commit in small steps:

1. **Fix short tags first, by hand or with PHP-CS-Fixer (`full_opening_tag` rule).** Analysis tools parse `<?` as plain text when `short_open_tag` is off, so they would miss code inside short-tag blocks.
2. **Report problems with PHPCompatibility** (a PHP_CodeSniffer standard). It lists removed and deprecated features by file and line. Use the 10.x release (still in alpha as of November 2026), because 9.x predates PHP 8. Run it with `--runtime-set testVersion 7.4-`, the range the code is moving *from*, not `8.2-`. Some changes are only reported when the range includes the version where they happened: with `8.2-`, it does not flag the PHP 4-style constructors at all, or the PHP 8.1 change to the default flags of `htmlspecialchars()`.
   - *Result on the development system (November 2026, PHPCompatibility 10.0.0-alpha2).* On the code before the fixes, it reported the 4 `each()` calls and the `${var}` interpolations. It reported the 8 constructors only with a 7.x range. After the fixes, the only findings are 45 `htmlspecialchars()` / `htmlentities()` calls without explicit flags. Since PHP 8.1 these also escape single quotes and substitute invalid UTF-8 instead of returning an empty string. All 45 escape text for display, so the new default is safe (and closes a gap for values placed in single-quoted attributes), and no change is needed.
3. **Apply the fixes automatically with Rector**, using its `UP_TO_PHP_82` level set (or a later one). It includes rules for PHP 4-style constructors to `__construct()`, `while (list(...) = each(...))` to `foreach`, and `${var}` to `{$var}` interpolation.
4. **Make the behavior changes by hand:** PDO error mode, loose comparisons, `count()` on non-arrays and input type checks. Tools cannot reliably detect or fix these, which is why the manual test checklist (1.1) matters.

**Target version:** PHP 8.2's security support ends on December 31, 2026. Making the code clean for 8.2 should be treated as the minimum, and it should be *deployed* on 8.3 or later. A code scan found no idioms affected by the breaking changes in 8.3, 8.4 or 8.5, so this should cost little or no extra work. The PHPCompatibility report (step 2) confirmed this.

#### 1.4 Low-Cost Fixes While the Code Is Open

These are small and independent, and they address problems listed earlier without changing the architecture:

- **Webhook:** verify GitHub's `X-Hub-Signature-256` (HMAC-SHA256) header and compare with `hash_equals()`.
- **Passwords:** switch to `password_hash()` and `password_verify()`, re-hashing each user's password the next time they log in successfully, so no one has to reset a password.
- **Front-end libraries:** move Bootstrap 3.3.7 to 3.4.1 (a drop-in release that fixes known XSS issues) and jQuery to the latest 3.x release. Remove the retired `google.com/jsapi` loader, the 404ing Google CSE stylesheet link, and the unused Bootstrap 3.2.0 files.
- **Broken links:** run the W3C Link Checker before and after Phase 1. Change the five `ieee-isto.org/privacy-policy/` links (`site.php` ×2, `pwg-cookie-notice.js`, `ipp/iana-registry.xsl` ×2) to `https://isto.org/privacy-policy/`, fix the `/3d` link in `index.php`, and fix or remove `search.html` (point it at the navbar search, or embed the current `cse.js`). Then do a recursive run and fix the broken links it finds inside individual pages. Record the remaining ones for Phase 3, which adds automated link checking.
- **`wrap.php`:** fix the JPEG pattern (`jpg|jpeg`) and remove the `X-UA-Compatible: IE=9` header.
- **Document uploads:** replace plain-FTP uploads with an encrypted method (FTPS or SFTP). If the archive ends up on the same host as the site, write the files locally instead.

#### 1.5 Deploy

- **Until production moves, every change must still run on PHP 5.4.** Merging to `master` deploys `dynamo/` to the Linode site, which runs PHP 5.4.16, through the webhook. So the Phase 1 code has to work on both 5.4 and 8.x. In particular, avoid PHP 7 syntax such as `??`, scalar type declarations and return types. Check with PHPCompatibility using `testVersion 5.4-` before each merge. Once production runs PHP 8, this limit goes away.
- Deploy to the staging environment and work through the test checklist.
- Confirm the hosting points listed in Problems, section 6: whether the `Action` directive works on the host (if not, an `.htaccess` `RewriteRule` that sends `.html` requests to `wrap.php` is a likely substitute), how deployment from GitHub will work without a server-side `git pull`, and where the FTP archive is served from.
- Switch production over, keeping the Linode VM available as a fallback until the new setup has run cleanly for an agreed period.

**Exit criteria:** every item on the test checklist passes on PHP 8.3 or later with no deprecation warnings logged, and production runs on that version.

### Later Phases

*To be defined once the requirements and open questions are settled.* Likely candidates:

- **Phase 2: Complete the move to managed hosting.** Close the remaining ICDSoft gaps, move the FTP archive at its existing URLs, and retire the Linode VM. If Open Question 12 is decided in favor, adopt the `www/` repository layout as part of setting up the new host. Mailing list remediation may run in parallel.
- **Phase 3: Restructure the site.** Separate static content from the web applications (Open Question 1), modernize page templating and navigation so they no longer need PHP edits, and add automated checks such as link checking and HTML validation.
- **Phase 4: Replace or retire individual web applications** (errata, IPP Everywhere listing, document registry) according to the decisions on Open Questions 2, 3 and 5.

## References

- **[BCP14]** S. Bradner, "Key words for use in RFCs to Indicate Requirement Levels", RFC 2119 / BCP 14, March 1997, https://www.rfc-editor.org/info/bcp14
- **[PWG-COMM-POLICY]** "PWG Communications Policy", PWG Policy (status: Approved), May 1, 2023, https://ftp.pwg.org/pub/pwg/general/process/pwg-comm-policy.pdf
- **[PWG-COMM-POLICY-DRAFT]** "PWG Communications Policy", draft PWG Policy (status: Initial), May 10, 2021 (document dated April 26, 2021), https://ftp.pwg.org/pub/pwg/general/wd/wd-pwg-comm-policy-v1.0-20210510.docx (earlier PDF: https://ftp.pwg.org/pub/pwg/general/wd/wd-pwg-comm-policy-v1.0-20210426.pdf). Includes plans for future changes that were removed from the approved policy.
- **[PWG-DOC-MGMT-POLICY]** "PWG Document Management Policy", PWG Policy (status: Approved), June 10, 2024, https://ftp.pwg.org/pub/pwg/general/process/pwg-document-management-policy.pdf
- **[PWG-PROCESS]** "PWG Process 4.0", PWG Process (status: Approved), August 4, 2023, https://ftp.pwg.org/pub/pwg/general/process/pwg-process-4.pdf
