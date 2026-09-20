# shadowaitools

**Shadow AI inventory from DNS, proxy and firewall exports.** Two implementations of the same tool, one for Node.js and one for Python, that read a log export on your own machine, reduce it to unique registrable domains, look each domain up once against a database of 20,000+ classified AI tool domains, and write out which AI tools are in use on the network, by whom, how often, and whether the vendor trains on the input.

| | Node.js | Python |
|---|---|---|
| Install | `npm install shadowaitools` | `pip install shadowaitools` |
| Command | `npx shadowaitools scan export.csv` | `shadowaitools scan export.csv` |
| Library | `const { scan } = require('shadowaitools')` | `from shadowaitools import scan` |
| Dependencies | none | `requests` |
| Runtime | Node.js 14+ | Python 3.7+ |
| Registry page | [npmjs.com/package/shadowaitools](https://www.npmjs.com/package/shadowaitools) | [pypi.org/project/shadowaitools](https://pypi.org/project/shadowaitools/) |

The lookup runs against the [AI tool classification database](https://www.aitoolsblocklist.com) at aitoolsblocklist.com. The hosted audit at [shadowaitools.com](https://www.shadowaitools.com) takes the same export in a browser and adds the per-user breakdown, dated vendor training verdicts, the sanctioned split and a PDF evidence pack.

This repository holds both packages: `node/` and `python/`.

---

## Contents

- [The problem this solves](#the-problem-this-solves)
- [What you get back](#what-you-get-back)
- [Log formats](#log-formats)
- [Node.js usage](#nodejs-usage)
- [Python usage](#python-usage)
- [Lookups, cost and caching](#lookups-cost-and-caching)
- [Privacy model](#privacy-model)
- [Hosted audit](#hosted-audit)
- [Related projects](#related-projects)
- [FAQ](#faq)
- [Links](#links)

## The problem this solves

Every organisation with a network has staff using AI tools that nobody approved. Chatbots for drafting, code assistants in the IDE, meeting transcribers that join calls, image generators for slides, file converters that accept a PDF and return a summary. Each one has a domain, each one receives content, and most of them say something in their terms about what they do with that content, or say nothing at all.

The usual answers are a survey, which people answer selectively, or an endpoint agent, which is a project. The cheaper answer is the log. A DNS filter, a web proxy or a firewall already records every hostname the network reached. The only missing step is classification: which of those thousands of hostnames belong to AI tools, what kind, and what the vendor does with the input.

That classification step is what `shadowaitools` adds. It is the equivalent of a [shadow AI discovery tool](https://www.shadowaitools.com) that you can run from a terminal, on a file you already have, without touching the network.

## What you get back

A JSON inventory with three parts.

**`summary`**: how much of the export was usable (`lines`, `records`), how many unique domains it contained, how many lookups were made, how many AI tools were found, how many users or devices were involved, and three training counters: `training_default_yes` (vendors that train on input unless you opt out), `training_no` and `unstated`.

**`by_category`**: tool counts per functional category, from Text & Language and Code & Development to Image & Visual, Audio, Voice & Music, Video, Agents & Automation, Search, Knowledge & Docs, Productivity & Collab and Models & Infrastructure.

**`tools`**: one object per AI tool found:

```json
{
  "domain": "otter.ai",
  "hosts": ["otter.ai", "app.otter.ai"],
  "hits": 7,
  "users": [{ "name": "laptop-sales-09", "hits": 4 }, { "name": "desktop-support-03", "hits": 3 }],
  "sanctioned": false,
  "primary_category": "Audio, Voice & Music",
  "categories": [{ "category": "Audio, Voice & Music", "subcategory": "Meeting transcription & notes" }],
  "ai_type": "ai_native",
  "trains_on_data": "yes",
  "opt_out_available": "unstated",
  "enterprise_no_training": "yes",
  "api_no_training": "unstated",
  "terms_checked": "2026-09-17"
}
```

`ai_type` separates services whose product is AI (`ai_native`) from ordinary products that added AI features (`ai_enabled`). Both receive pasted content, so both are listed, and the flag lets a policy treat them differently. The four training fields take the values `yes`, `no`, `opt_out_default` and `unstated`, and `terms_checked` is the date the vendor's terms were last read.

Both packages also render the inventory as CSV (one row per tool, users and hosts joined with semicolons) and as a fixed-width table for terminals, tickets and chat.

## Log formats

The first non-empty line of the file decides the parser.

| Format | Recognised by | Examples |
|---|---|---|
| `csv` | header row with a hostname column: `domain`, `query`, `QueryName`, `hostname`, `host`, `url`, `dest`, `destination`, `fqdn`, `site`, `sni`, `question` | Cisco Umbrella, Cloudflare Gateway, DNSFilter, NextDNS, Palo Alto URL log, Zscaler web log, any spreadsheet export |
| `key-value` | `hostname=`, `dstname=`, `url=`, `srcip=` tokens | Fortinet FortiGate, SonicWall, other syslog-style firewalls |
| `dnsmasq` | `query[A] name from client` | Pi-hole, dnsmasq |
| `squid` | native `access.log` layout with `CONNECT host:443` | Squid |
| `windows-dns` | debug log packets with `(3)www(6)openai(3)com(0)` names | Windows DNS Server |
| `generic` | anything else | plain hostname lists, URL lists, ad hoc logs |

The user, device or client column is optional and detected by name (`Identities`, `user`, `Source User`, `DeviceName`, `device_name`, `client_ip`, `src`, `cip`, `email` and similar). Without it the inventory still has tools, hosts and hit counts. Delimiters may be comma, tab, semicolon or pipe, and when the hostname cell holds a full URL the host is extracted from it.

Hostnames in reserved zones (`.local`, `.internal`, `.lan`, `.corp`, `.home`, `.arpa`, `.test`, `.example`) are dropped before any lookup.

## Node.js usage

```bash
npm install shadowaitools
export SHADOWAITOOLS_API_KEY=your_key
npx shadowaitools scan umbrella.csv --csv out.csv --json out.json --sanctioned openai.com,github.com
```

```js
const { scan, parseLog, extractDomains, toTable } = require('shadowaitools');

// Dry run: format and lookup count, no network
const parsed = parseLog('umbrella.csv');
console.log(parsed.format, parsed.lines, extractDomains(parsed.records).length);

// Full scan with a cache so tomorrow's run only pays for new domains
const inventory = await scan('umbrella.csv', {
  apiKey: process.env.SHADOWAITOOLS_API_KEY,
  cacheFile: '/var/lib/shadowaitools/lookups.json',
  sanctioned: ['openai.com', 'github.com'],
  onProgress: (done, total) => process.stderr.write(`${done}/${total}\r`),
});
console.log(toTable(inventory));
```

A Playwright, Express or Electron project can embed the parser to build an upload page that classifies exports for a whole team without any file leaving the company:

```js
const express = require('express');
const multer = require('multer');
const { scan } = require('shadowaitools');

const app = express();
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 25 * 1024 * 1024 } });

app.post('/scan', upload.single('export'), async (req, res) => {
  const inventory = await scan(req.file.buffer, {
    apiKey: process.env.SHADOWAITOOLS_API_KEY,
    cacheFile: './lookups.json',
  });
  res.json(inventory);
});
app.listen(3000);
```

## Python usage

```bash
pip install shadowaitools
export SHADOWAITOOLS_API_KEY=your_key
shadowaitools scan fortigate.log --csv out.csv --json out.json
shadowaitools domains fortigate.log       # lookup count before spending anything
```

```python
from shadowaitools import scan, to_table

inv = scan("fortigate.log", cache_file="lookups.json", sanctioned=["openai.com", "github.com"])
print(to_table(inv))

for t in inv["tools"]:
    if t["trains_on_data"] in ("yes", "opt_out_default") and not t["sanctioned"]:
        print(t["domain"], t["hits"], "hits by", ", ".join(u["name"] for u in t["users"]))
```

A pandas workflow:

```python
import pandas as pd
from shadowaitools import scan

inv = scan("zscaler.csv")
df = pd.DataFrame(inv["tools"])
df["users_n"] = df["users"].apply(len)
print(df.groupby("primary_category")[["hits", "users_n"]].sum().sort_values("hits", ascending=False))
print(df[df["trains_on_data"].isin(["yes", "opt_out_default"])][["domain", "hits", "users_n", "terms_checked"]])
```

## Lookups, cost and caching

One lookup per unique registrable domain. Subdomains collapse first, so `api.openai.com`, `chat.openai.com` and `platform.openai.com` cost one lookup between them. A 400-line sample export with 45 distinct domains uses 45 lookups; a month of resolver logs with two million lines typically reduces to a few thousand.

Both packages read and write the same JSON cache file, a map of domain to lookup result. Run the Node CLI on Monday and the Python script on Tuesday against the same cache and the second run makes no request for any domain the first one saw. The `domains` command in either package prints the format and the exact lookup count without making a request.

The lookup itself is `GET https://www.aitoolsblocklist.com/api/check?domain=<domain>` with an `X-API-Key` header, the same endpoint the [`aiblocklist`](https://www.npmjs.com/package/aiblocklist) client exposes directly. A response carries `quota_remaining`, which the inventory summary reports.

## Privacy model

- The export is parsed on the machine that runs the command. It is never uploaded by these packages.
- Only registrable domains are sent, one per lookup. No URLs, no paths, no query strings, no timestamps, no user names.
- Reserved and private zones are filtered out before the lookup step.
- The cache holds lookup results keyed by domain and nothing from the log lines.

The hosted audit at shadowaitools.com follows the same idea one step further: the upload is parsed once, matched, and discarded; reports stay in the account for 90 days and can be deleted earlier.

## Hosted audit

The packages give you the inventory. Management, auditors and clients usually want a report. The hosted audit, reachable as [shadow AI tools](https://www.shadowaitools.com) at shadowaitools.com, takes the same export and produces:

- every AI tool found with category, AI type and risk level
- the per-user, per-device or per-IP breakdown
- a dated training verdict for each vendor
- the sanctioned versus unsanctioned split against your approved list
- sector policy verdicts (Block, Controls, Allow) from the AI Policy Profiles
- CSV export and a PDF evidence pack

The free preview names a fifth of the tools found and comes with a preview PDF. Full reports are one-time purchases without a subscription; the subscription plans on aitoolsblocklist.com include one to ten audits a month.

## Related projects

| Package | Registry | What it wraps |
|---|---|---|
| `aiblocklist` | [npm](https://www.npmjs.com/package/aiblocklist), [PyPI](https://pypi.org/project/aiblocklist/) | the lookup, feed and stats endpoints of the AI tools blocklist, the database this tool resolves against |
| `aitoolsblocklist` | [npm](https://www.npmjs.com/package/aitoolsblocklist), [PyPI](https://pypi.org/project/aitoolsblocklist/) | the same API, original client |
| `aiagentallowlist` | [npm](https://www.npmjs.com/package/aiagentallowlist), [PyPI](https://pypi.org/project/aiagentallowlist/) | the [page-type database for AI agents](https://www.aiagentallowlist.com), the AI agent allow list that decides per URL what a browsing agent may open, 40 million+ domains, up to 28 page types each |
| `webfilteringdatabase` | [npm](https://www.npmjs.com/package/webfilteringdatabase) | the [web filtering database](https://www.webfilteringdatabase.com), 120M+ domains in 59 categories |
| `websitecategorization` | [npm](https://www.npmjs.com/package/websitecategorization), [PyPI](https://pypi.org/project/websiteclassificationapi/) | the [website categorization API](https://www.websitecategorizationapi.com), 700+ IAB content categories |
| `phishingdetectionapi` | [npm](https://www.npmjs.com/package/phishingdetectionapi), [PyPI](https://pypi.org/project/phishingdetectionapi/) | the [phishing detection API](https://www.phishingdetectionapi.com), 390,000+ DNS-verified active phishing domains |
| `cipawebfiltering` | [npm](https://www.npmjs.com/package/cipawebfiltering), [PyPI](https://pypi.org/project/cipawebfiltering/) | [CIPA web filtering](https://www.cipawebfiltering.com) for schools and libraries |

The [AI tools blocklist](https://www.aitoolsblocklist.com) is the natural next step after an inventory: the same 18 categories, shipped as EDL, PAC, hosts and DNS feeds for the filter you already run, so the tools you decide to block can be blocked by category the same afternoon. Mirror of this repository: [gitlab.com/url-classifications/shadowaitools](https://gitlab.com/url-classifications/shadowaitools).

## What to do with the inventory

The list usually sorts itself into three piles. Tools that are already approved go on the sanctioned list, so the next run flags only what is new. Tools in categories the organisation has decided against, such as deepfake generators, voice cloning or AI companions, go to the network filter by category, which is what the AI tools blocklist feeds are for. Everything in between, typically the chatbots, transcribers and code assistants that a team adopted because they work, goes to a conversation about an enterprise plan, an opt-out setting or a replacement, with the `trains_on_data` column and its `terms_checked` date as the starting evidence. Repeating the scan monthly and diffing the tool lists shows drift, and the cache keeps that repeat almost free.

## Why inventory first

The [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) orders its functions Govern, Map, Measure, Manage, and Map is where an organisation lists the AI systems it actually uses. [Shadow IT](https://en.wikipedia.org/wiki/Shadow_IT) research has said for years that unsanctioned tools are found in traffic, not in interviews. The [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) names sensitive information disclosure as a leading risk, which for an organisation means knowing which services staff paste into. The [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) places asset inventory under Identify for the same reason. A DNS log is the most complete asset list a network has; this tool reads it.

## FAQ

**Which file do I export?**
A week of DNS or web traffic from whatever already sees it: the DNS filter's activity export, the proxy's web log, the firewall's URL or web filter log. CSV with a header row is ideal, syslog and dnsmasq logs work as they are.

**How many AI tools should I expect?**
On a network of a few hundred people, a week of traffic usually reaches somewhere between thirty and two hundred AI tool domains, most of them never mentioned to IT. The sample exports in this repository, built from ten synthetic users, produce 33 to 34.

**Does it work without a user column?**
Yes. Tools, hosts and hit counts come from the hostname column alone. The per-user breakdown needs a user, device or client column, which most filter exports include.

**Can I run it on a schedule?**
Yes. Both CLIs exit non-zero on errors, write JSON and CSV, and share a cache, so a cron entry or a scheduled task after the nightly export is the common setup. The examples above include a nightly script and a weekly report.

**What is the API key?**
An AI Tools Blocklist key from the account area at aitoolsblocklist.com. Plans include a monthly lookup quota; `summary.quota_remaining` shows what is left after a scan.

**Is this the same as blocking AI tools?**
No. This is discovery. Blocking is what the AI tools blocklist feeds are for, and the inventory tells you which categories are worth blocking on your network and which tools to sanction instead.

## Links

- Hosted shadow AI audit: https://www.shadowaitools.com
- AI tools blocklist: https://www.aitoolsblocklist.com
- AI agent allow list: https://www.aiagentallowlist.com
- NIST AI Risk Management Framework: https://www.nist.gov/itl/ai-risk-management-framework
- NIST Cybersecurity Framework: https://www.nist.gov/cyberframework
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- Shadow IT: https://en.wikipedia.org/wiki/Shadow_IT

## License

MIT. Copyright Alpha Quantum, info@alpha-quantum.com.
