# LintPage Action

Audit SEO, accessibility, and performance on preview deployments before they reach production.

This GitHub Action runs [LintPage](https://lintpage.com) audits on your pages and reports issues as GitHub annotations on your pull request's checks.

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `urls` | URLs to audit, separated by spaces or newlines. Each URL is audited separately | No | The `deployment_status` target URL |
| `config` | Path to a LintPage config file | No | `.lintpagerc.json`, `.lintpagerc` or `lintpage.config.json` in the working directory |
| `fail-on-warning` | Fail the check on warnings (not just critical issues) | No | `false` |
| `quiet` | Minimal output: the verdict and notices plus critical annotations (and warnings with `fail-on-warning`) | No | `false` |
| `vercel-bypass-secret` | Vercel Protection Bypass for Automation secret, passed from a GitHub secret. Sent only to the `*.vercel.app` URLs this action audits. See below | No | |
| `version` | `lintpage` CLI version to install: an npm version or dist-tag. Pinned so a CLI release cannot change your results unannounced; set `latest` to always use the newest | No | `0.5.8` |

The config file is read from the checked-out repository, so run `actions/checkout` before this action. A missing or invalid config file falls back to the default rules without an error.

## Outputs

| Output | Description |
|---|---|
| `audited` | Number of URLs audited. Skipped entries are not counted |
| `exit-code` | Worst result across all URLs: `0` pass, `1` findings, `2` error or a skipped entry |

## Upgrading

`@v1` points at the latest v1 release.

### v1.2.6

- The CLI is pinned to `0.5.8`. It adds findings and changes some, so a check that passed can now fail and one that failed can now pass. To keep the 0.5.7 results, set `version: '0.5.7'`; that also keeps the memory and time problems described below. To soften one finding instead, set its severity or `false` in your config, as for any rule (`page-truncated` and `page-too-complex` included): turning off `page-truncated` hides only the notice, and the findings a cut page skips stay skipped; turning off `page-too-complex` lets a page whose content was not checked pass. New findings lower the score of a page that has them, a critical one much more than an info one; the scoring weights are unchanged. None of the rules that fetch run against a local build (`localhost`). The "Upgrading from 0.5.7" section of the [CLI README](https://www.npmjs.com/package/lintpage) has every detail.
- New findings that can fail the check:
  - `title`: critical when the title is only the audited site's domain name (`example.co.uk`), a warning when it is that domain with a generic page word (`Contact | example.ro`); the warning "Missing title tag" when the only `<title>` sits in an inline SVG. On a preview deployment the domain is the preview's host, so a title that names only the production domain is not flagged there but is on production.
  - `page-too-complex` (new): critical when checking the page before it is built needs more than 4 seconds of CPU time in all, or runs out of stack, which the depth limit makes unlikely (below).
  - `content-language` (new, LP-061): a warning when `lang` or `og:locale` names another language than the page text.
  - `image-alt-text`: a warning when images use only the audited site's domain name as alt text, except logos (an image with "logo" in its or its parent's class, id, alt or source, or one that links to the home page) and images with `role="presentation"` or `role="none"`.
  - `sitemap-presence`: a warning when the sitemap robots.txt names on your site fails to load (a 401 or 403 is only info) or is not a valid sitemap, or when robots.txt names sitemaps only on other hosts and one of them is on localhost or an IP address.
- New info findings, which never fail the check: `page-truncated` (new: only part of the page was checked), `sitemap-coverage` (new, LP-062: pages linked from this page are missing from the sitemap), `cors-policy` (the page did not answer within 15 seconds, so its CORS headers were not checked) and `sitemap-presence` (the named sitemap refused LintPage with 401 or 403, or sits on another host).
- Findings that can go away, so a check that failed can now pass:
  - on a page cut at 5 MiB or at a parse limit, "missing" findings and findings about content past the cut;
  - `target-unreachable` on a page nested so deeply that 0.5.7's parser ran out of stack, such as thousands of nested `<template>` elements: it is now cut at the depth limit and checked up to it;
  - `sitemap-presence`: the findings about `/sitemap.xml` ("No sitemap.xml found", not valid XML, empty, could not be fetched, timeout) when robots.txt names a sitemap on your site, which is checked instead; "may not be valid XML" for RSS, Atom, text and gzipped sitemaps; a broken `/sitemap.xml` when robots.txt names sitemaps only on other hosts (now info, or a warning when one of them is on localhost or an IP address);
  - `cors-policy`: its findings, on a page that does not answer within 15 seconds.
- The audited page is read up to 5 MiB (uncompressed). Before, the CLI read the whole response: an endless response grew it past 3.8 GB, and a 2 MB gzip response that inflates to 2 GiB crashed it. A page of 5 MiB or more gets the info `page-truncated`, and findings that look for something that may sit past the cut are not reported: no H1, no `<main>` or no semantic elements, no JSON-LD (and the invalid-JSON warning for the last block, which the cut may have split), no analytics. A robots `noindex` past the cut is not seen, so that critical finding can go away too. When the cut goes through the `<head>`, its "missing" findings (title, meta description, canonical, viewport, favicon, Open Graph, Twitter card, hreflang) are not reported either, and `thin-content` and `content-to-html-ratio` skip the page.
- Before the page is built, a first parse checks it within 4 seconds of CPU time and against caps on what makes parsing slow (250,000 elements, 600,000 nodes, total nesting depth, 100 frames and others). On 0.5.7, 18 KB of misnested formatting tags took 176 seconds and 3.4 GB to parse. A page over a cap is checked up to that point, as a cut page, and `page-truncated` names the cap. A page whose checks need more than 4 seconds in all gets the critical `page-too-complex`: its content is not checked (no page checks, no `broken-internal-links` or `sitemap-coverage`), while `robots-txt-check`, `sitemap-presence`, TTFB, CORS and the AI crawler checks still run on a public URL. The 4 seconds are CPU time, so a page close to the limit can pass on a fast machine and fail on a slow runner; on Node before 22.19 (or 23.0 to 23.8), the CPU time of the CLI's other threads, such as garbage collection, also counts. The action runs the CLI on the runner's Node, or on the one an earlier `actions/setup-node` step installs. Building the page afterwards has no time limit. The largest real pages measured, the single-page HTML and ECMAScript specifications (each cut at 5 MiB), need well under the 4 seconds.
- `content-language` detects the language of the visible text and compares it with the `lang` of `<html>` (or `<body>`) and `og:locale`, primary subtag only. It says nothing on pages with little text, when the declared language holds 10% or more of the text, or when the text is spread over more than two languages, so multilingual pages pass. Languages the detector mixes up, such as Danish and Norwegian or Czech and Slovak, count as one.
- `sitemap-presence` checks the first sitemap robots.txt names on the audited site and reads `Sitemap:` lines as Google does. When robots.txt names none, or does not load, it checks `/sitemap.xml` as before. For a named sitemap the messages name its URL, an error status (other than 401 or 403) or a failed fetch is a warning, and 401 or 403 is info, as on Wikipedia, which limits its sitemap to approved crawlers. Sitemaps named only on other hosts are not fetched. RSS, Atom and text sitemaps are accepted, and a gzipped sitemap is recognised and not checked further. robots.txt and the sitemap share one 30-second limit, and the audit makes one more request, for robots.txt.
- `sitemap-coverage` reads robots.txt and the sitemaps on the audited site (up to 11 files, 20 MiB) and fetches up to 5 of the linked pages it does not find there: at most 17 requests, within 20 seconds. A page is reported only when it answers 200 with HTML, is indexable and is allowed for Googlebot; when a sitemap cannot be read in full, or one sits on another host, nothing is reported. URLs are compared by path and query, so a preview deployment whose sitemap names the production domain still matches. It is info for now: on large sites it also lists pages left out of the sitemap on purpose, such as login and search pages.
- `ttfb-pro` and `cors-policy` give the page 15 seconds to answer and read only its headers. Before, the HTTP client waited up to 300 seconds. When the headers take longer, `ttfb-pro` reports the warning "The server did not respond within 15 seconds, so TTFB could not be measured", with the same severity as 0.5.7's slow-response warning, and `cors-policy` reports the info finding above in place of what late headers showed. A server that does not complete the connection within 10 seconds still hits the HTTP client's connect timeout and gives the old messages.
- No change to results: robots.txt rules are prepared once per file. A large robots.txt of non-ASCII rules cost over a second of CPU per audit; it is now well under a second.
- Output only: the CLI no longer prints the page's CSS parse errors to the log, which could run to hundreds of megabytes for one crafted page.

### v1.2.5

- The CLI is pinned to `0.5.7`. robots.txt, sitemap.xml and the AI crawler probes now have time limits, described below. A site that misses one can get findings it did not get before, or lose some, so a check that passed can now fail and one that failed can now pass. A probed page of 5 MiB or more no longer gets the warning `ai/empty-content-to-bot`, so with `fail-on-warning` a check that failed only on it can now pass. Other sites get the same results. To keep the 0.5.6 results, set `version: '0.5.6'`; that also brings back the stalls described below.
- robots.txt, sitemap.xml and the AI crawler probes can no longer stall the job or run it out of memory. `robots-txt-check` and the AI crawler checks give robots.txt 5 seconds to start answering and 10 seconds to arrive in full. `sitemap-presence` gives sitemap.xml 30 seconds and reads at most 50 MiB, the size limit Google applies; a larger sitemap is checked on its first 50 MiB only. Each AI crawler probe gives the page 15 seconds in all and reads at most 5 MiB; the headers still have to arrive within 5 seconds, as before. Before, `robots-txt-check` and `sitemap-presence` set no time limit of their own, and the AI checks stopped waiting only for the headers. robots.txt was already read only up to 500 KiB. The audited page itself and the other rules that fetch (`ttfb-pro`, `cors-policy`, `broken-internal-links`) fetch as before. None of these rules run against a local build (`localhost`), so a local build sees no change.
- When robots.txt or sitemap.xml does not arrive in time, `robots-txt-check` or `sitemap-presence` reports a warning that says so, in place of its other findings. With `fail-on-warning`, that fails the check. A slow robots.txt that blocks Googlebot is no longer a critical finding, so without `fail-on-warning` such a page can now pass. If your robots.txt is generated per request (for example `app/robots.ts` on a preview deployment), check that it starts answering well within 5 seconds.
- The AI crawler checks already treated robots.txt as missing when its headers took more than 5 seconds; they now also do when it starts answering in time but has not arrived in full within 10 seconds. Then findings that come only from its rules are not reported: `ai/all-crawlers-blocked` for a site that disallows every AI bot, `ai/*-user-blocked` for a bot robots.txt disallows, and `ai/*-extended-disallowed` (the first two still fire when the live fetch is blocked). A bot that robots.txt disallows and your server answers with 403, 429 or a 5xx then gets the critical `ai/*-silent-block` finding (GPTBot, ClaudeBot, PerplexityBot), which fails the check; a bot it allows already got it on 0.5.6.
- The near-empty content check (`ai/empty-content-to-bot`) skips a probe that runs out of time and a page cut at 5 MiB, so with `fail-on-warning` a page that failed only on it can now pass. A probe that runs out of time is not counted as a block, as before.
- No change to results: a robots.txt with many `User-agent` lines no longer slows the job down. 0.5.6 took seconds to parse a file with a few hundred `User-agent` lines followed by rules up to the 500 KiB limit, and with thousands of `User-agent` lines tens of seconds and gigabytes of memory, which could crash the CLI; the action reports that as an error.
- No change to results: the AI crawler checks measure the text of a probed page in linear time. Before, a page with many unclosed `<` or `<script` tags took time that grew with the square of its size (64 KiB of `<` took about 5 seconds).

### v1.2.4

- The CLI is pinned to `0.5.6`. Some rules report more and a few report less, so a check that passed can now fail and one that failed can now pass. To keep the 0.5.5 results, set `version: '0.5.5'`.
- `robots-noindex` reads every robots meta tag, not only the first, and `<meta name="googlebot">` too, with names and directives in any letter case. Directives count only as whole words, separated by commas, spaces or semicolons, and `none` counts as `noindex, nofollow`. A page with `content="NOINDEX"`, a second robots meta carrying `noindex`, or a googlebot `noindex` now gets the critical finding; the nofollow warning follows the same rules. Content that only contains the word, such as `noindex|nofollow`, no longer matches, so a page that failed only on that can now pass.
- `robots-txt-check` decides about the audited page itself, not only a site-wide `Disallow: /`. A block for Googlebot is critical. A block only in the `User-agent: *` group, while a `Googlebot` group allows the page, is a warning: where 0.5.5 did not report such a block, the warning is new and fails with `fail-on-warning`; where it was a `Disallow: /` in the `*` group, it drops from critical to a warning, so without `fail-on-warning` a page that failed on it can now pass. `Disallow: /blog` on a /blog page now fails, while `Disallow: /` with `Allow: /blog/` no longer reports /blog/post as blocked. When only the query string is blocked, the message says so without showing the query string. Setting a severity for `robots-txt-check` is not a way to keep the old results: it applies to every `robots-txt-check` finding, so lowering it would also stop a whole-site `Disallow: /` from failing.
- The robots.txt parser behind that check and the AI crawler checks matches the path plus query string as Google documents, reads `Googlebot/2.1` as `Googlebot`, handles CR-only line endings and non-ASCII paths, ignores rules before the first `User-agent` line and past the first 500 KiB, and can no longer be stalled by a hostile robots.txt. This works both ways: a rule it used to miss can now report an AI crawler as blocked, and a rule it used to apply can stop matching, for example `Disallow: /*.php$` on a URL with a query string. A bot that robots.txt now disallows no longer gets the critical `ai/*-silent-block` finding.
- `anchor-missing-href` counts a `javascript:` href as missing. With `fail-on-warning`, a page with such a link can now fail.
- Output only, no change to results: the job summary lists every critical and warning finding (up to 50 per page), then info findings while the table has fewer than 10 rows; before, it listed the 10 most severe. With `quiet`, it no longer points at the log for findings `quiet` leaves out of it. Every URL of a run shares one step summary, and if it nears GitHub's 1 MiB limit, a page's section is replaced by a one-line note so GitHub does not drop the whole summary.

### v1.2.3

- The CLI is pinned to `0.5.5`. To keep the 0.5.4 results, set `version: '0.5.4'`.
- `cors-policy` reports `Access-Control-Allow-Origin: *` without credentials as info instead of a warning. Vercel (and Cloudflare Pages on `*.pages.dev`) send it on static pages by default, and without credentials it only lets other sites read what anyone can already fetch. With `fail-on-warning`, a page that failed only on this can now pass. A wildcard with credentials stays critical, and a server that reflects any origin stays a warning, or critical with credentials. Setting a severity for `cors-policy` in your config is not a way to keep the old warning: it applies to every `cors-policy` finding, so it would also turn the critical cases into warnings. For the same reason, if you lowered or disabled `cors-policy` to quiet the Vercel warning, remove that override now: it also hides the critical cases.
- The CLI package now carries the license notices for the code it bundles (undici and llhttp).

### v1.2.2

- The CLI is pinned to `0.5.4`. Two rules change, as below. To keep the 0.5.3 results, set `version: '0.5.3'`.
- `meta-render-blocking` counts only scripts a browser runs as classic JavaScript. Scripts held back by a consent manager (`type="text/plain"`) or moved to a worker by Partytown (`type="text/partytown"`) are no longer counted as blocking, nor are scripts browsers never run: a `type` with parameters such as `text/javascript; charset=utf-8`, a non-JavaScript `language` attribute, or an empty `src`. With `fail-on-warning`, a page that failed only on this rule can now pass.
- `tracking` reads only the Content-Security-Policy directives that let a page load or connect, such as `script-src`, `connect-src` and `default-src`. An analytics host named only in `report-uri`, `frame-ancestors` or `form-action` no longer counts, so a page whose only sign of analytics was such a host now gets the info `tracking` finding. If you raised `tracking` to a warning as described under v1.2.1, that page can now fail with `fail-on-warning`; set its severity back to `INFO` to avoid it.

### v1.2.1

- The CLI is pinned to `0.5.3`. Two rules report less, so with `fail-on-warning` a page that failed only on them can now pass. To keep the 0.5.2 results, set `version: '0.5.2'`.
- `tracking`: a page with no analytics visible in its HTML gets an info finding instead of a warning, since analytics loaded from a JavaScript bundle cannot be seen. An analytics host in a `preconnect`, `dns-prefetch`, `preload` or `modulepreload` link, or in a Content-Security-Policy header or meta tag, now counts as found. To keep the warning, set `"tracking": { "enabled": true, "severity": "WARNING" }` under `rules` in your config file.
- `meta-render-blocking` no longer counts a `<script nomodule>`, such as the Next.js polyfills, as blocking.

### v1.2.0

- The CLI is pinned to `0.5.2`. A page that redirects to another host, including a Vercel deployment that redirects to the Vercel login page, is now reported by the CLI as a critical `redirected-off-site` finding (exit code `1`), rather than refused by the action (exit code `2`). Such a URL now counts in `audited`.
- New `vercel-bypass-secret` input to audit deployments behind Vercel Deployment Protection. See [With Vercel preview deployments](#with-vercel-preview-deployments).
- On a `*.vercel.app` URL, the `X-Robots-Tag: noindex` Vercel adds to deployment URLs is reported as info, not as a critical issue.
- The `quiet` input is back and works.
- Annotations name the page they are about, show `:` and `,` as written, and each audited URL adds a section to the job summary.
- If you set `version` below `0.5.2`, a protected Vercel deployment can pass again by auditing the Vercel login page; v1.1.1 refused it in the action.

### v1.1.1

A `*.vercel.app` deployment behind Vercel Deployment Protection used to pass, because the audit checked the Vercel login page instead of your site. It now fails with an error that names the cause, and is not counted in `audited`. Nothing else changes.

### From v1.0.0 to v1.1.0

Three changes can affect an existing workflow:

- Every URL in `urls` is audited. v1.0.0 audited only the first URL and silently skipped the rest, so a job that passed before can now fail on a URL that was never checked.
- The CLI is pinned to `0.5.1` instead of `latest`. To use a newer CLI, set `version`, or wait for an action release that raises the default.
- The `api-key` input has been removed. The CLI no longer accepts the `--report` flag it enabled, so setting it made every run fail. GitHub now ignores it with an "Unexpected input" warning; delete it from your workflow.

## Usage

### With Vercel preview deployments

```yaml
name: LintPage
on: deployment_status

jobs:
  lintpage:
    if: github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: lintpage/action@v1
```

This runs for every successful deployment, production included. To audit previews only, also check `github.event.deployment.environment` in the `if:`.

#### Deployments behind Vercel Deployment Protection

If your project uses Deployment Protection, the deployment URLs Vercel reports redirect to the Vercel login page, for production deployments as well as previews. Without access, the audit fails with a critical `redirected-off-site` finding that names the cause. To audit the deployments:

1. In Vercel, open the project's **Settings**, then **Deployment Protection**, and create a **Protection Bypass for Automation** secret.
2. In GitHub, open the repository's **Settings**, then **Secrets and variables**, then **Actions**, and add it as a repository secret named `VERCEL_AUTOMATION_BYPASS_SECRET`.
3. Pass it to the action:

```yaml
      - uses: lintpage/action@v1
        with:
          vercel-bypass-secret: ${{ secrets.VERCEL_AUTOMATION_BYPASS_SECRET }}
```

The secret reaches the CLI through its environment, never on a command line, and neither the action nor the CLI prints it. Pass it from `secrets`, as above: GitHub masks secrets in logs, but a plain string or a `vars` value given as an input is shown in the step's log before the step runs. It is sent as the `x-vercel-protection-bypass` header only to the https `*.vercel.app` URLs this action audits (from `urls` or the deployment event), and only to that exact origin: not to other sites the audit contacts, and not along a redirect to another host. With the secret set, list only your own project's deployments in `urls`, since every audited `*.vercel.app` URL receives it.

Use the `vercel-bypass-secret` input rather than setting `VERCEL_AUTOMATION_BYPASS_SECRET` in the step's `env`: the CLI also needs the list of URLs it may send the secret to, which only the input sets up, so the environment variable alone sends nothing.

Vercel also sends `X-Robots-Tag: noindex` on its deployment URLs, so the CLI reports that noindex as info on a `*.vercel.app` URL, and says so. Audit your production domain to check its indexing. Previews may also get a `canonical-self-referencing` warning when their canonical tag points at the production domain, which matters if you set `fail-on-warning`.

### With explicit URLs

```yaml
name: LintPage
on: pull_request

jobs:
  lintpage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: lintpage/action@v1
        with:
          urls: 'https://staging.myapp.com https://staging.myapp.com/pricing'
```

### With local build

```yaml
name: LintPage
on: pull_request

jobs:
  lintpage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: npm ci
      - run: npm run build
      - run: npm start > server.log 2>&1 &
      - run: npx wait-on -t 60000 http://localhost:3000
      - uses: lintpage/action@v1
        with:
          urls: 'http://localhost:3000'
```

Against `localhost`, LintPage skips the checks that only make sense for a public site and lowers some others to info. A notice in the output lists both. Other private addresses, such as a service container at `http://web:3000`, are refused.

### With custom config and a pinned CLI

```yaml
- uses: lintpage/action@v1
  with:
    urls: 'https://staging.myapp.com'
    config: './seo/lintpage.config.json'
    fail-on-warning: 'true'
    version: '0.5.8'
```

## How it works

1. Installs the `lintpage` CLI at the version set by `version`
2. Takes URLs from the `urls` input or, if it is empty, from the `deployment_status` event's target URL. Fails with an error if there is neither
3. Audits each URL with `--format github`, in its own log group. Every URL is audited even if an earlier one fails
4. Fails the step if any URL has critical issues (or warnings with `fail-on-warning`), or makes the CLI error. A redirect to another host, such as a Vercel login page, is a critical issue. To report without blocking, set `continue-on-error: true` on the step and read the `exit-code` output

Findings appear as annotations in the run summary and on the pull request's Checks tab, not inline on the diff. Each finding's annotation names the page it is about. GitHub shows at most 10 annotations of each severity per step, and every URL runs in this one step, so with many URLs or findings some are left out. Each URL's verdict is a notice that comes before that URL's findings, but it shares the 10 notices with info findings and the other URLs' verdicts, so with several URLs a later verdict can be left out; `quiet` drops the info annotations and leaves those notices to the verdicts. Each URL also adds a section to the job summary with its verdict, score, counts, every critical and warning finding (up to 50) and, while there is room, info findings; `quiet` does not shorten it. The log has every finding unless `quiet` is on, which leaves out the findings that do not fail the check.

## License

MIT
