---
layout: post
title: PostgreSQL日志换行处理
categories: PostgreSQL
---

<!--more-->

## PostgreSQL日志Line Break处理

	$ cat logfile | awk '{if ($0 !~  "^[\t]") printf("\n%s" ,$0); else printf $0}'
	$ cat logfile | perl -e 'while(<>){ if($_ !~ /^\t/ig) { chomp; print "\n",$_;} else {chomp; print;}}'
