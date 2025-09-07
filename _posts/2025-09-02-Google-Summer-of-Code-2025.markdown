---
layout: default
title: "Google Summer of Code 2025 - Reengineering ZIMFarm from the Ground Up"
date: 2025-09-01 06:00:00 +0100
author: "Uchechukwu Orji"
permalink: /gsoc-2025/
header_buttons: '<a href="https://github.com/openzim/zimfarm" class="button" target="_blank">View Project on GitHub</a>'
---

In the summer of 2025, I participated [again](/gsoc-2024/) in <a href="https://summerofcode.withgoogle.com/" target="_blank">Google Summer of Code</a> with
<a target="_blank" href="https://kiwix.org/">Kiwix</a> reengineering the
<a href="https://github.com/openzim/zimfarm" target="_blank">Zimfarm project</a>. The ZIM farm (zimfarm)
is a semi-decentralized software solution to build <a href="https://wiki.openzim.org/wiki/ZIM_file_format" target="_blank">ZIM files</a> efficiently. This means scraping web contents,
packaging them into a ZIM file and uploading the results to an online ZIM files repository.

### About the Organization

<a href="https://kiwix.org/" target="_blank">Kiwix</a> is a non-profit organization and a free and open-source software project dedicated to providing
offline access to free educational content. By compressing copies of entire websites into a single
<a href="https://wiki.openzim.org/wiki/ZIM_file_format" target="_blank">ZIM file</a>
such that they can fit on a user's device, it provides <a href="https://kiwix.org/en/applications/" target="_blank">applications</a>
that can read these local copies, thus, enabling people with no or limited internet access to enjoy the same browsing experience as anyone else.

### Project Details

At a high level, the project comprises:

- **Backend API** - a REST API that manages recipes and distributes tasks.
- **Frontend UI** - a web UI to create/edit recipes and monitor tasks/workers.
- **Workers** - machines that run scrapers in containers to build ZIM files, then upload the results.
- **Uploader & Receiver** - handle file transfers and validate ZIM before publishing.
- **Scrapers** - conainerized tools (e.g for MediaWiki, Stack Exchange, Gutenberg) that convert websites into ZIM format.

### Work Done

As of September 1st, 2025, the reengineering of the ZIMFarm project spanned
<a href="https://github.com/openzim/zimfarm/pulls?page=4&q=is%3Apr+author%3Aelfkuzco+is%3Aclosed+merged%3A%3E%3D2025-05-22" target="_blank">over 110 pull requests</a>,
with Python and TypeScript serving as the primary programming languages used in development.
The code for the project lives on <a href="https://github.com/openzim/zimfarm" target="_blank">openzim/zimfarm</a>.

Starting from the backend, I began with the introduction of <a href="https://hatch.pypa.io/" target="_blank">Hatch</a>
as a dependency manager to pin all dependencies to a specific version. Some of the libraries were upgraded to major versions while some were replaced entirely with more feature-rich ones. The most notable replacements in this ambitious reengineering were:

- Flask &rarr; FastAPI
- Marshmallow &rarr; Pydantic
- Paramiko and `subprocess.run` calls &rarr; Cryptography library
- JavaScript &rarr; TypeScript
- Vue 2 &rarr; Vue 3

As a consequence of these upgrades and replacements, a few breaking changes were introduced (it was inevitable to keep them away).
This meant the versioning of the backend API to **v2**, not only to take full advantage of features from library replacements but also
to clean up parts of the old API that were fragile and inelegant such as:

- relying on `subprocess.run` calls to perform actions such as verification of authentication messages
- query parameters that contained special characters crashing server (<a href="https://github.com/openzim/zimfarm/issues/1131" target="_blank">#1131</a>)
- crashes in user creation when fields that should be required (like an email address) were missing <a href="https://github.com/openzim/zimfarm/issues/1058" target="_blank">#1058</a>
- UI inconsistencies that caused buttons to remain active even when no changes were pending (<a href="https://github.com/openzim/zimfarm/issues/994" target="_blank">#994</a>)
- ZIM metadata values not being escaped properly (<a href="https://github.com/openzim/zimfarm/issues/1203" target="_blank">#1203</a>)

Aside from the library switches and upgrades, the reengineering introduced significant features including but not limited to:

- improved security by properly escaping flag inputs when constructing offliner commands (<a href="https://github.com/openzim/zimfarm/pull/1216">#1216</a>)
- improved task assignment by introducing context-based filtering, ensuring tasks run only on compatbile workers (<a href="https://github.com/openzim/zimfarm/pull/1233" target="_blank">#1233</a>)
- added support for SSH keys generated using the ECDSA algorithm (<a href="https://github.com/openzim/zimfarm/pull/1190" target="_blank">#1190</a>, <a href="https://github.com/openzim/zimfarm/pull/1195" target="_blank">#1195</a>)
- addded support for SSH keys generated using the Ed25519 signature scheme (<a href="https://github.com/openzim/zimfarm/pull/1279" target="_blank">#1279</a>)
- standardized schedule language codes to the ISO 639-3 format (<a href="https://github.com/openzim/zimfarm/pull/1241" target="_blank">#1241</a>)
- switched from bare bones `requirements.txt` to hatch for dependency management (<a href="https://github.com/openzim/zimfarm/pull/1106" target="_blank">#1106</a>)
- used modern type annotations and tooling like <a href="https://github.com/microsoft/pyright" target="_blank">Pyright</a>, and
  <a href="https://docs.astral.sh/ruff/" target="_blank">Ruff</a>
  to enforce type-checking and code quality
- introduced functions to introspect Pydantic schemas and Python type definitions, thus, enabling the extraction of type information for validation and client-side reuse (<a href="https://github.com/openzim/zimfarm/pull/1150" target="_blank">#1150</a>, <a href="https://github.com/openzim/zimfarm/pull/1246" target="_blank">#1246</a>)
- enforced ZIM metadata conventions (<a href="https://github.com/openzim/zimfarm/pull/1224" target="_blank">#1224</a>)
- improved the UI by making it more responsive, appealing and usable on small/mobile screens

_To avoid turning the list into a long and boring changelog, I've highlighted only a few select improvements (not ranked in any way). If you are
curious, you can browse the full list of pull requests on <a href="https://github.com/openzim/zimfarm/pulls?page=4&q=is%3Apr+author%3Aelfkuzco+is%3Aclosed+merged%3A%3E%3D2025-05-22" target="_blank">Github</a>)_

I tried to keep the UI largely the same as the orignal (partly because I am not good at design 😅) and only made ambitious changes in the UI where
necessary. Relying heavily on <a href="https://vuetifyjs.com/" target="_blank">Vuetify</a>, I gave the UI a more modern design and introduced some additional features and pages.

### Challenges

Reengineering a project of this size was by no means a small feat and it was challenging as much as it was exciting.
The biggest challenges I faced (in no particular order) during
the project revolved around converting Marshmallow models to Pydantic models,
extracting metadata from Pydantic models to be able to share validation rules to clients, relaxing validation
while reading existing data from the database, but enforcing at writes. A lot of this meant
I had to <a href="https://docs.pydantic.dev/latest/concepts/validators/" target="_blank">wrap around validators</a> to make them skip validation based on contexts.

To be able to extract metadata from the Pydantic models, there was quite a lot of instrospection
using the <a href="https://docs.python.org/3/library/typing.html#" target="_blank">typing module</a> (more on that in a later blog post).

Similarly, the project featured <a href="https://docs.pydantic.dev/latest/concepts/fields/#discriminator" target="_blank">Pydantic's discriminators</a> to be able to discriminate between the
different schemas of the various scrapers/offliners.

The frontend served as a learning guide for me to use Vue 3 and TypeScript. Prior to this, I had
only used them sparingly (I cannot remember the last time I did something frontend-related). But
after the first couple of weeks, I began to find my feet, thanks to foundational knowledge in
JavaScript.

There were some minor changes in the database schema which meant I had to employ <a href="https://www.postgresql.org/docs/9.5/functions-json.html" target="_blank">PostgreSQL JSON functions</a>
to be able to get the functionality I needed.

### Future Work

We plan to wrap up this GSoC project with the deployment of the `zimfarm-upgrade` branch on September 8th, 2025. Of course, the journey doesn't end
here. There's still plenty of work ahead with issues of different priorities ranging from
<a href="https://github.com/openzim/zimfarm/issues?q=is%3Aissue%20state%3Aopen%20label%3Aprio1" target="_blank">prio1</a>,
<a href="https://github.com/openzim/zimfarm/issues?q=is%3Aissue%20state%3Aopen%20label%3Aprio2" target="_blank">prio2</a>, and
<a href="https://github.com/openzim/zimfarm/issues?q=is%3Aissue%20state%3Aopen%20label%3Aprio3" target="_blank">prio3</a> to a broader
backlog waiting to be tackled.

I’m glad to share that I’ll continue working with Kiwix as a contractor until at least the end of 2025, which is both a testament to their trust
in the work I’ve done and to how much I’ve developed since my first pull request to their codebase two years ago.

### Things I Learned

Reengineering the project exposed me to more situations where I had to make decisions regarding
backwards-compatibility and the trade-offs associated with it.

Navigating the challenges mentioned earlier meant I had to introspect heavily in order to be able
to come up with an elegant solution while still maintaing strict type-checking standards. Also, I
learnt newer things about SQL, most importantly the <a href="https://www.postgresql.org/docs/9.5/functions-json.html" target="_blank">JSON functions</a> and how to wrangle data to in the database.

If last year's participation in the Google Summer of Code changed the way I wrote code, this year's edition deepened my engineering discipline and how I think about building systems.

### Acknowledgements

I express gratitude to Google for providing me with this opportunity to contribute to Open Source Software for the second time in a row.

Thanks to the team at Kiwix for reviewing my pull requests during the submission phase with the same responsiveneess, accepting my proposal and
making this a reality.
If you are a newbie or a seasoned developer looking to get started with open-source and collaborative development, I implore you to start with the
Kiwix codebase. The team is incredibly responsive, offers constructive feedback, and makes the contribution process both welcoming and rewarding.
You’ll not only sharpen your technical skills but also get the chance to work on projects that make a real impact.

Most thanks of all goes to my project mentor <a href="https://github.com/benoit74" target="_blank">Benoît Béraud</a> for his feedback and help with
challenges during the project. Without his feedback, none of this would have been possible as
he almost always had a suggestion when I hit a wall. His careful organization of the issues
and detailed explanations meant there was a little to and fro on the issues, thus, accelerating
the rate of development.

Working with him significantly improved my approach to problem-solving.
