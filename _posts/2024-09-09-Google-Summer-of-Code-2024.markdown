---
layout: default
title: "Google Summer of Code 2024 - An Automated Download Speed Testing Solution"
date: 2024-09-09 14:48:49 +0100
author: "Uchechukwu Orji"
permalink: /gsoc-2024/
header_buttons: '<a href="https://github.com/kiwix/mirrors-qa" class="button" target="_blank">View Project on GitHub</a>'
---

In the summer of 2024, I participated in <a href="https://summerofcode.withgoogle.com/" target="_blank">Google Summer of Code</a> with
<a target="_blank" href="https://kiwix.org/">Kiwix</a> developing an
<a href="https://summerofcode.withgoogle.com/programs/2024/projects/kr1RiKiJ" target="_blank">Automated Download Speed Testing</a>
solution that tests the download speed of users to Kiwix download mirrors from various locations on Earth. The goal of the project
was to provide the organization with data and analytics that would help them re-configure
<a href="https://mirrorbrain.org/" target="_blank">MirrorBrain</a> to direct users to the _"best"_
mirrors for their region. Overall, this would ensure fast and reliable download speeds for all users.

### About the Organization

<a href="https://kiwix.org/" target="_blank">Kiwix</a> is a non-profit organization and a free and open-source software project dedicated to providing
offline access to free educational content. By compressing copies of entire websites into a single
<a href="https://wiki.openzim.org/wiki/ZIM_file_format" target="_blank">ZIM file</a>
such that they can fit on a user's device, it provides <a href="https://kiwix.org/en/applications/" target="_blank">applications</a>
that can read these local copies, thus, enabling people with no or limited internet access to enjoy the same browsing experience as anyone else.

### Project Details

At a high level, the project involved:

- setting up a Wireguard container for running speed tests from various locations across the Earth,
- recording the download speed metrics against a specific mirror from a specific location,
- storing the results in a PostgreSQL database and
- building a <a href="https://www.metabase.com/" target="_blank">Metabase dashboard</a> using SQL to answer questions
  that would help the organization re-configure MirrorBrain.

{:refdef: style="text-align: center;"}
![Project Overview](/assets/images/gsoc-2024-project-details.png)
{: refdef}

### Work Done

The project was a large 350-hour project that spanned
<a href="https://github.com/kiwix/mirrors-qa/pulls?q=is%3Apr+is%3Aclosed+author%3Aelfkuzco" target="_blank">over 20 pull requests</a>
with Python and SQL the primary languages used in development.
The code for the project lives on <a href="https://github.com/kiwix/mirrors-qa" target="_blank">kiwix/mirrors-qa</a>.


In order to avoid tight-coupling of logic, the project was split into three core services namely:

- **backend**: This service, written in Python, comprises:
  - **REST API server**: A web server built with FastAPI that exposes endpoints for viewing and submission of test results.
  - **Scheduler**: A script that periodically creates tasks for _"idle"_ workers.
  - **CLI**: An entrypoint that exposes sub-commands for registering workers and updating mirror metadata in the database.
- **worker**: Also written in Python, this service records the download speed metrics across mirrors from various locations
  and submits the results to the backend. It achieves this through the following components:
  - **task**: A service that downloads a file from a mirror, records the speed metrics such as latency, duration, etc.
    and writes results to disk.
  - **manager**: On receiving a task from the backend, this service communicates with the Docker daemon and:
    - sets up a Wireguard container with the necessary configuration for the test country as set by the scheduler.
    - starts the task service as a container.
    - configures the task container to use the network stack of the Wireguard container.
    - binds its filesystem to that of the task service.
    - reads results and submits to the backend web server.
- **dashboard**: This service builds up a Metabase dashboard with SQL queries to answer questions including but not limited to:
  - Which regions/countries are slow to download from for a particular mirror?
  - What are the top mirrors for a region/country?
  - How does the _current_ speed within a period compare to the median speed before the period?

All services were containerized using Docker, making it easy to set up and deploy.

### Challenges

The biggest challenges I faced during the project revolved around the limitations of the open source version of Metabase.
At the time of writing, it didn't have support for charts like the 
<a href="https://github.com/metabase/metabase/issues/10783" target="_blank">Box Plot</a>and the 
<a href="https://github.com/metabase/metabase/issues/13799" target="_blank">Pivot table</a> didn't have support for
sticky columns.

Also, filter names could not be scoped to a specific tab in the dashboard. This made it difficult to have filters that shared
the same name in different tabs but didn't necessarily have to be connected.

Another challenge I had was developing a script to download all Wireguard configuration files from ProtonVPN. I hacked
around it by digging through their core libraries and experimenting with their ABC classes. It was a thrilling experience and I couldn't
have done it without <a href="https://github.com/BurntSushi/ripgrep" target="_blank">ripgrep</a> to scan for _"helpful"_ 
variable names and methods across their core libraries.

### Future Work

At the time of writing, there is a plan to
<a href="https://github.com/kiwix/mirrors-qa/issues/37" target="_blank">automatically retrieve metabase data</a>,
process it and use it to configure MirrorBrain.

### Things I Learned

Working on the project made me become more proficient with industry tools like
<a href="https://docs.github.com/en/actions" target="_blank">GitHub Actions</a>
and <a href="https://docs.docker.com/" target="_blank">Docker</a>. Prior to the project, I had only
used some of their well-known features for my hobby projects.

Also, I learnt a lot more about SQL than I had ever used before. Features like 
<a href="https://www.postgresqltutorial.com/postgresql-window-function/" target="_blank">window functions</a>
and even <a href="https://www.postgresqltutorial.com/postgresql-plpgsql/postgresql-create-function/" target="_blank">creating custom functions</a>
were things I had never used before but they ended up playing a crucial role in the achievement the goal of the project.

### Acknowledgements

I express gratitude to Google for providing me with this opportunity to contribute to Open Source Software
and develop my skills at the same time.

Thanks to the team at Kiwix for reviewing my pull requests during the submission phase, accepting my proposal and
making this a reality.

Most thanks of all goes to my project mentor <a href="https://github.com/rgaudin" target="_blank">Renaud Gaudin</a> for his feedback and help with
challenges during the project. It was a great experience working with him and I always looked forward to his code reviews.
Working with him helped me improve my coding style and approach to problem-solving.
