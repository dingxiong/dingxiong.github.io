---
layout: post
title: All I know about Markdown
date: 2025-04-20 12:58 -0700
categories: [productivity]
tags: [markdown]
---

I was confused why prettier formats my todo lists "wrongly" constantly. Then I
read this [issue](https://github.com/prettier/prettier/issues/17305) and found
it is my problem. The canonical description of markdown syntax does not specify
the syntax unambiguously, so we need a standard. The most referenced standard
is [spec.commonmark](https://spec.commonmark.org/0.31.2/).

Section 5.2 covers all details about lists. Quoted below.

> The most important thing to notice is that the position of the text after the
> list marker determines how much indentation is needed in subsequent blocks in
> the list item. If the list marker takes up two spaces of indentation, and
> there are three spaces between the list marker and the next character other
> than a space or tab, then blocks must be indented five spaces in order to
> fall under the list item.

Most time, the continuation blocks are indented at least to the column of the
first character other than a space or tab after the list marker. If the list
mark is a dash with a following space, then the sublist should have 2 space
indentation. If the list mark is `1.` with a following space, then the sublist
should have 3 space indentation. If the list mart is `10.` with a following
space, then the sublist should have 4 space indentation.

After understanding this rule, now I can understand how prettier reformats my
todo lists.
