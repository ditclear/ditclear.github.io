---
layout: post
title:  "flask sqlite3.OperationalError: unable to open database"
date:   2015-4-26
---

<p class="intro"><span class="dropcap">学</span>习Flask的时候遇到了sqlite3.OperationalError: unable to open database的问题。查了一下，觉得主要的问题是自己的路径写成了绝对路径，导致了错误。</p>

##解决办法
```python
	import os

	PROJECT_ROOT = os.path.dirname(os.path.realpath(__file__))

	DATABASE = os.path.join(PROJECT_ROOT, 'tmp', 'test.db')
```
