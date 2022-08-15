---
title: {{ replaceRE "[0-9]{2,}" "" .Name | replaceRE "^-*" "" | replaceRE "-" " " | title }}
date: {{ .Date }}
lastmod:
author: Gonejack

description:
categories: []
tags: []

draft: false
enableDisqus : false
enableMathJax: false
toc: false
---
