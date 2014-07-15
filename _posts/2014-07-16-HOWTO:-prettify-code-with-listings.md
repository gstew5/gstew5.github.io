---
layout: post
title: HOWTO Prettify Coq Code with Listings
published: False
---

[Listings](http://www.ctan.org/pkg/listings) makes typesetting code easy. The following are some (personal) tips for producing pretty listings for Coq:

Use a sans serif, nonmonospaced font, by setting
```
basicstyle=\sffamily
```
and 
```
columns=fullflexible.
```
For C, assembly, etc., use 
