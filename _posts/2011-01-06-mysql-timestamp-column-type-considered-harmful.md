---
layout: post
title: MySQL timestamp column type considered harmful
description: On one of MySQL's weirder column types, and why you shouldn't use it.
---

<a href="https://www.flickr.com/photos/telstar/433029904" title="Time Selector by Todd Lappin, on Flickr"><img src="https://farm1.staticflickr.com/174/433029904_27dd44a95a.jpg" width="500" height="381" alt="Time Selector"></a>

MySQL has a weird auto-updating field type called `TIMESTAMP`, which can be automatically populated or updated with the current time when you insert or update a row. It's probably not 100% kosher practice to have a system changing your data without your explicit approval, and yet one could imagine this being useful for logging purposes, or for simplifying your insertion queries.

Unfortunately, this auto-updating functionality operates under very awkward and seemingly pointless constraints. The details are laid out in the [MySQL 5.0 reference manual](http://dev.mysql.com/doc/refman/5.0/en/timestamp.html), but the upshot is like this:

- Regardless of how many `TIMESTAMP` fields your table contains, only one of those fields may be automatically populated or updated with the current time.
- The first `TIMESTAMP` field will have the auto-populating, auto-updating functionality enabled *by default*, and the only way to disable it is to force a default value on the field.

In other words, you have very little flexibility in terms of which fields can use this feature, and you could have automatic changes to your data that are not obvious in your schema unless you are well-versed in the idiosyncracies of MySQL's time types.

This behavior has evolved only slightly over the past decade, and in such a way as to complicate and exacerbate the problem rather than fix it. With MySQL 4.1, you gained the ability to apply the auto-update feature to a `TIMESTAMP` field other than the first, but the process of doing so involves setting a default value on the first `TIMESTAMP`, a pointless and potentially damaging restriction.

Moreover, you are still forced to contain all automatic behavior to a single field. This is fine in the case of automatic updates, since any auto-updating timestamp beyond the first would be redundant. But it also applies to auto-populating, which means you cannot have a field that automatically populates itself with the time of insertion (a creation timestamp) alongside a `TIMESTAMP` field that automatically updates itself when the row is updated (an update timestamp).

I'm sure the longevity of this "feature" is due to backwards-compatibility reasons, with so many extant schemas in the wild relying on it, but it really ought to be deprecated and highlighted in the documentation with red 24-point Impact and &lt;blink&gt; tags.

For keeping time data, I use the `DATETIME` type almost exclusively in my schemas. It obviates the issue by providing no automatic behavior whatsoever. If I use `TIMESTAMP` at all, it only occurs once in a table, and it is always used to track when a row was updated. I even mark it with the verbosely redundant attributes `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, which might be pedantic or paranoid or both, but at least one can tell how the field will behave when glancing at the schema.
