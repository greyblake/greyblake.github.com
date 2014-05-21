---
layout: post
title: "Ignore files with git locally"
date: 2014-05-21 16:37
comments: true
categories: git rails
---

Sometimes it's necessary to ignore some files in a repository only locally.
For rails developers it's often `./config/database.yml` file.
Every developer has his own database configuration.

With git it can be easily achieved, we may instruct git no to track changes in
certain files:

```bash
git update-index --assume-unchanged ./config/database.yml
```

Next time we type `git status` the changes in `./config/database.yml` won't be shown.

If you think you need to track that file again, just do:

```bash
git update-index --no-assume-unchanged ./config/database.yml
```

(pay attention to `--no` prefix).

Thanks!

