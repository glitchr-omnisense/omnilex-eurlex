# omnilex/eurlex

EU law and the case law of the Court of Justice for [omnisense/omnilex](https://github.com/glitchr-omnisense/omnilex),
from two open accesses of the Publications Office of the European Union to Cellar, the repository
behind EUR-Lex (<https://op.europa.eu/web/cellar/cellar-data/metadata/knowledge-graph>), both
without a key:

| Access | Address | Used for |
|---|---|---|
| the SPARQL endpoint | `https://publications.europa.eu/webapi/rdf/sparql` (`EurlexSource::SPARQL_URI`) | the metadata, described with the Common Data Model (CDM): titles, dates, whether an act is in force, its consolidated versions, the links between acts and judgments; every search |
| the resources by CELEX number | `https://publications.europa.eu/resource/celex/<CELEX>` (`EurlexSource::RESOURCE_URI`) | the texts: content negotiation, `Accept: application/xhtml+xml, text/html;q=0.9` and `Accept-Language: <fra>` redirect to the XHTML of the document in that language |

EUR-Lex's own search web service, for registered users, is not used. There is no sandbox: the
endpoint is open.

## Installation

```sh
composer require omnilex/eurlex
```

Requires PHP 8.2, `omnisense/omnilex` (1.x) and `symfony/http-client` (6.4, 7 or 8). In a Symfony
application with `Omnilex\Bridge\Symfony\OmnilexBundle`, the factory is registered as soon as the
package is installed.

## Credentials

None: both accesses are open. Calls carry the `User-Agent` of the `user_agent` option.

## Options

| Option | Type | Default | Required | |
|---|---|---|---|---|
| `language` | ISO 639-1 | `fr` | no | the language of the titles, labels and texts: one of the 24 official languages of the Union, mapped to the Office's codes (`fr` → `FRA`); another is an `InvalidConfigException` when the source is built, which `Registry::usable()` leaves out |
| `content` | bool (`"false"`, `"0"` false) | `true` | no | whether `text()` and `decision()` also fetch the document (one more call, up to a few megabytes); `article()` always does |
| `throttle` | float | `0.5` | no | seconds between two calls |
| `base_uri` | string | `https://publications.europa.eu/webapi/rdf/sparql` | no | the SPARQL endpoint |
| `resource_uri` | string | `https://publications.europa.eu/resource/` | no | |
| `user_agent` | string | `omnilex/1.x (+https://github.com/glitchr-omnisense/omnilex)` | no | |

Without an HTTP client, the factory makes one with a 60-second timeout.

## Capabilities

`capabilities()`: holds texts, articles and decisions; reads CELEX, ELI and ECLI; criteria `text`,
`title`, `kind`, `jurisdictions`, `from`, `to`, `subjects`, `types`; jurisdictions `EU`; versions
yes; 100 per page; `pseudonymised` false (the Office does not say).

| Operation | Supported | Reads |
|---|---|---|
| `search()` | yes, the titles and metadata (not the full texts) | |
| `text()` | yes, as in force at a date | CELEX (sector 3 and others but 6), ELI of EU law (`http://data.europa.eu/eli/...`) |
| `article()` | yes, of an act divided into articles, with its number | CELEX or ELI of the act, and the article's number |
| `decision()` | yes | ECLI of the EU courts (`ECLI:EU:...`), CELEX of sector 6 |
| `citations()` | yes, both ways | CELEX, ELI, ECLI |
| `recent()` | yes, by the date of the document | |

## Behaviour

**A work.** Every reader first asks the endpoint for the work behind the identifier: its CDM
properties (CELEX number, ELI, ECLI, sector, dates of the document, of signature, of entry into
force and end of validity, in force or not, consolidation date, resource type, subject matters,
author, court formation, procedure, Official Journal) with their labels in the source's language,
and its title. A work that answers to several CELEX numbers (`62012CJ0131`, `62012CJ0131_1`) is
read by the plain one. Unknown: `null`.

**An act at a date.** An act is published in the Official Journal (CELEX `32016R0679`), then
consolidated each time it is amended or corrected (CELEX `02016R0679-20160504`: the state of the
text on 4 May 2016). `Text::$versions` lists them: the act as published from its date (label
`Official Journal`), then each consolidated version from its date to the day before the next
(label `consolidated <date>`); the last one runs to the act's end of validity. A consolidation
dated the day the act starts to apply replaces the published text from that day. `text()` and
`article()` read the version that covers the day asked for (today by default):

- a day before the act's date: `null`;
- a day between its date and its first entry into force: the version, with `Status::FUTURE`;
- a day after its end of validity (a repealed directive): `null`;
- the last version says whether the act is in force (`resource_legal_in-force`): `IN_FORCE`;
  `REPEALED` when it is not and its end of validity is past; `EXPIRED` when it is not otherwise;
  `UNKNOWN` when the Office does not say; the earlier ones are `MODIFIED`;
- a consolidated CELEX number names its version: the day is not read, the version's status is
  `UNKNOWN`.

A version's `from` is the day the text took that wording. The dates of entry into force and of
application are in `Text::$raw['resource_legal_date_entry-into-force']`. `Text::$publishedOn` is
`null`; `date` is the date of the document.

**A text.** `text()` of a judgment (sector 6) is a `NotSupportedException`. The `Text` has the
CELEX number as `id`, its title, its CELEX, ELI and ECLI identifiers, the resource type (`REG`,
`DIR`...) as `type`, the subjects' labels, the XHTML of the version read as `html` and as plain
`content` (with `content: true`), `language`, the EUR-Lex page as `url`.

**An article.** The Office publishes recent acts as XHTML divided into `eli-subdivision` blocks.
`article()` takes the block `art_<number>` of the version of the day (`bis`, `ter`, `quater` read
as `a`, `b`, `c`; then the number without its punctuation): its heading as `title`, its text
without the heading as `content`, the block as `html`, `<celex of the version>#art_<n>` as `id`,
the EUR-Lex page with the anchor as `url`. An article that is not there is `null`; an act whose
text is not divided into articles in that language is a `NotSupportedException`; `article()`
without a number too.

**A decision.** `decision()` of an act (not sector 6) is a `NotSupportedException`. The title
EUR-Lex gives (`Arrêt de la Cour (grande chambre) du 13 mai 2014.#Google Spain SL ...#Demande de
décision préjudicielle ...#Affaire C-131/12.`) is cut at its `#`: the first two parts are the
`title`, the third the `summary`; the case numbers (`C-131/12`) are read from it, the first as
`number`. `court` is the authoring corporate body (`Cour de justice`, code `CJ`) with the court
formation as `formation`; `citations` the links from the judgment; `solution`, `publication` are
`null`.

**Search and what is new.**

| `Query` | SPARQL |
|---|---|
| `text`, `title` | `bif:contains` on the titles in the source's language (both set: the words of both, all wanted): every word of two characters or more (`"a" AND "b"`), or the exact expression in double quotes; a text with no such word is an `InvalidArgumentException` |
| `kind` | `TEXT`: sector 3 (legislation); `DECISION`: sector 6 (judgments, orders, opinions of the Advocates General - `type` says which); none: both; `ARTICLE`: `NotSupportedException` |
| `types` | `work_has_resource-type`: `REG`, `DIR`, `DEC`, `REG_IMPL`, `JUDG`, `ORDER`, `OPIN_AG`... |
| `subjects` | `resource_legal_is_about_subject-matter`: the codes of the subject-matter table (`PROT`, `ENV`...) |
| `jurisdictions` | `work_created_by_agent`: `CJ` (Court of Justice), `GCEU` (General Court); `EU` means any |
| `from`, `to` | `work_date_document` |
| `sort` | by date, then CELEX number: newest first, or `oldest`; `relevance` is read as newest |
| `limit`, `cursor` | 100 a page at most; the cursor is a row offset; no total |

A code of `types`, `subjects` or `jurisdictions` with other characters than letters, digits, `_`,
`.` and `-` is an `InvalidArgumentException`. Summaries and information notes (CELEX numbers
with `_`) are left out. `recent($since)` is the same query from that day, newest first; it reads
the date of the document, not the day it was added to Cellar: a document published late with an
old date is not listed as new.

**Links.** `work_cites_work` (`CITES`), `case-law_interpretes_resource_legal` (`INTERPRETS`),
`resource_legal_based_on_resource_legal` (`BASED_ON`), `resource_legal_amends_resource_legal`
(`AMENDS`), `resource_legal_repeals_resource_legal` and its implicit form (`REPEALS`), and their
inverses when read from the document cited. `citations($id, $limit)` gives the outgoing links then
the incoming ones, at most `$limit` (1 to 1000) each way, newest first; targets without a CELEX
number are left out; a target with no title in the source's language is titled with its CELEX
number.

## Example

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Omnilex\Eurlex\EurlexSourceFactory;
use Omnilex\Model\Kind;
use Omnilex\Model\Query;
use Symfony\Component\HttpClient\HttpClient;

$eurlex = (new EurlexSourceFactory(HttpClient::create()))->create(['language' => 'fr', 'content' => false]);

$results = $eurlex->search(new Query(text: 'protection des données', kind: Kind::TEXT, types: ['REG'], from: '2016-01-01', limit: 5));
foreach ($results as $reference) {
    echo $reference->date?->format('Y-m-d'), ' ', $reference->id, ' ', mb_substr($reference->title, 0, 90), "\n";
}
echo 'next: ', var_export($results->next, true), "\n\n";

$decision = $eurlex->decision('ECLI:EU:C:2014:317');
echo $decision->title, "\n", $decision->court?->name, ' / ', $decision->court?->formation, ', ', $decision->date?->format('Y-m-d'), ', ', $decision->number, ', ', $decision->ecli(), "\n";
foreach (array_slice($decision->citations, 0, 3) as $c) {
    echo '  ', $c->relation->value, ' ', $c->target->id, ' ', mb_substr($c->target->title, 0, 70), "\n";
}
echo "\n";
$text = $eurlex->text('32016R0679');
echo count($text->versions), " versions:\n";
foreach ($text->versions as $v) {
    echo '  ', $v->id, ' ', $v->from?->format('Y-m-d'), ' -> ', $v->to?->format('Y-m-d') ?? '...', ' ', $v->status->value, ' (', $v->label, ")\n";
}
```

```
2026-05-20 32026R1165 Règlement (UE) 2026/1165 du Parlement européen et du Conseil du 20 mai 2026 modifiant le r
2024-03-13 32024R0868 Règlement (UE) 2024/868 du Parlement européen et du Conseil du 13 mars 2024 modifiant la d
2019-03-25 32019R0493 Règlement (UE, Euratom) 2019/493 du Parlement européen et du Conseil du 25 mars 2019 modif
2018-10-23 32018R1725 Règlement (UE) 2018/1725 du Parlement européen et du Conseil du 23 octobre 2018 relatif à 
2018-05-23 32016R0679R(02) Rectificatif au règlement (UE) 2016/679 du Parlement européen et du Conseil du 27 avril 20
next: '5'

Arrêt de la Cour (grande chambre) du 13 mai 2014. - Google Spain SL et Google Inc. contre Agencia Española de Protección de Datos (AEPD) et Mario Costeja González.
Cour de justice / Grande chambre, 2014-05-13, C-131/12, ECLI:EU:C:2014:317
  cites 62012CJ0473 Arrêt de la Cour (troisième chambre) du 7 novembre 2013. - Institut pr
  cites 62012CJ0342 Arrêt de la Cour (troisième chambre) du 30 mai 2013. - Worten – Equipa
  cites 62009CJ0509 Arrêt de la Cour (grande chambre) du 25 octobre 2011. - eDate Advertis

2 versions:
  32016R0679 2016-04-27 -> 2016-05-03 modified (Official Journal)
  02016R0679-20160504 2016-05-04 -> ... in_force (consolidated 2016-05-04)
```

(called live on 2026-10-26 at 11:56 UTC, from the core's harness image)

In Symfony:

```yaml
omnilex:
    sources:
        eurlex: { factory: eurlex, options: { language: fr } }
```

From the core's harness: `docker compose run --rm omnilex article eurlex 32016R0679 17 --at 2018-06-01`.

## Errors

| Case | Result |
|---|---|
| an unknown CELEX, ELI or ECLI; an article not in the act; a day before the act or after its end | `null` (`citations()`: `[]`) |
| a French ECLI or ELI, another identifier | `NotSupportedException` (`The "eurlex" source cannot read a text "...".`) |
| `text()` of a judgment, `decision()` of an act, an article of an act not divided into articles, `article()` without a number, `kind: ARTICLE` in a search, a criterion outside the list | `NotSupportedException` |
| the endpoint down, a 5xx, a timeout | `UnavailableException` |
| a 429 | `RateLimitedException` |
| an answer without SPARQL results | `ProviderException` (`the SPARQL endpoint did not answer with results`) |

## Verified, and not

Tested offline: `Tests/EurlexSourceTest.php` (17 tests), on `MockHttpClient` with answers recorded
on 2026-10-04 - regulation 32016R0679 and its versions, its text as published and as consolidated
(cut down to a few articles), judgment ECLI:EU:C:2014:317 and its links, a search of the titles,
the judgments since 2026-09-24, a criterion refused, down or slow never an empty answer, an
unknown language.

Called live: on 2026-10-04 (regulation 2016/679 with two versions; directive 2006/112/EC with 28
versions - 2008-06-01 gives `02006L0112-20071229`, 2019-03-03 gives `02006L0112-20160601`; the
repealed directive 95/46/EC - nothing today, `01995L0046-20031120` in 2010) and on 2026-10-26 (the
example above, article 17 of the GDPR at 2018-06-01). No limit is published for the endpoint; a
whole text weighs up to a few megabytes (directive 2006/112/EC: 1.9 MB): cache it - a version that
is no longer the last never changes.

## Responsibility

An independent, unofficial client, not affiliated with the Publications Office of the European
Union. Consolidated versions are documentary: only the Official Journal is authentic. Whoever runs
it complies with EUR-Lex's terms ([EUR-Lex's legal
notice](https://eur-lex.europa.eu/content/legal-notice/legal-notice.html)): see the README's
Responsibility section.
