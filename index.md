---
layout: default
title: AK // INFRASTRUCTURE LOG
author: Anthony Klein
description: Systems administration workflows and technical documentation.
---

### [ SYSTEM_STATUS: ONLINE ]

**Welcome** to my centralized technical repository. This space serves as a living document for my work in systems administration, network architecture, and automation engineering.

Whether it's deep-dives into Proxmox virtualization, CrowdSec implementations, or general infrastructure logic, this log is where I archive proven workflows and technical observations.

---

_**Note:** Entries dated prior to 2020 are maintained primarily for historical reference and should be considered **legacy documentation**. While these archives reflect past workflows, some external links or dependencies may be deprecated._

---

**Environment Specs:**
* **Engine:** [Jekyll](https://jekyllrb.com/) 
* **Host:** [GitHub Pages](https://github.com/KDN-Cloud/b.aklein.me)
* **Workflow:** GitOps / Static Deployment

---

<br>
### [ ARCHIVE_LOGS ]

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%b %d, %Y" }}</small>
{% endfor %}

