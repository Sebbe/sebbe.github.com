---
layout: post
title: EPiServer find custom tracking
category: episerver
description: We use Per Magne Skuseth briliant tool Dynamic multi search and to get statistics to work with that tool, we followed Henrik Fransas guide for how to do custom query and click tracking and all looked to work pretty well, but something was wrong when looking at specific languages.
tags: [episerver-find]
---

We use Per Magne Skuseth briliant tool [Dynamic multi search](https://world.episerver.com/blogs/Per-Magne-Skuseth/Dates/2015/9/dynamic-multi-search-for-episerver-find/) and to get statistics to work with that tool, we followed Henrik Fransas [guide for how to do custom query and click tracking](http://world.episerver.com/blogs/Henrik-Fransas/Dates/2015/2/how-to-do-custom-query-and-click-tracking-with-episerver-find/) and all looked to work pretty well, but something was wrong when looking at specific languages.

It tracked click and queries correct as you can see in this image
![find tracking]({{ site.baseUrl }}/images/2017-10-05/find_all_languages.png) but if you look at the top, then you will see that this is for "all websites" and "all languages". Lets change that to only be for norwegian and this is what we got ![find tracking norwegian]({{ site.baseUrl }}/images/2017-10-05/find_norwegian.png) It didn't track anything for the diffrent languages, so when we wanted to use best best etc then we couldn't set it up currectly

The code for the custom tracking I used was 
```csharp
public void TrackClick(string query, string hitId, string trackId)
{
    SearchClient.Instance.Statistics().TrackHit(query, hitId, command =>
    {
        command.Hit.Id = hitId;
        command.Id = trackId;
        command.Ip = "127.0.0.1";
        command.Tags = ServiceLocator.Current.GetInstance<IStatisticTagsHelper>().GetTags();
        command.Hit.Position = null;
    });
}
```
That we got from Henrik's blog post, and the parameter that decides what language it is, is the Tags parameter. 

We get the list width diffrent tags from Episervers ``` IStatisticTagsHelper.GetTags() ``` function and this is what I got ![find tags]({{ site.baseUrl }}/images/2017-10-05/tags.png)

This looked quiet wrong as I work on a single site, so it shouldn't be two sites there, so I reported this to Episerver Support since I thought this was a bug.

Turns out, this is not a bug. From what I understand from Support you should maybe not use that function at all to get the tags, but if you use it, then you need to send "false" as a parameter in the function ``` IStatisticTagsHelper.GetTags() ``` else it won't track correctly for the diffrent languages

Here is the full code from Henrik, but I have added false to the function so it works correctly. Now I need to find out if I should use the ``` IStatisticTagsHelper.GetTags() ``` or not.

```csharp
public void TrackClick(string query, string hitId, string trackId)
{
    SearchClient.Instance.Statistics().TrackHit(query, hitId, command =>
    {
        command.Hit.Id = hitId;
        command.Id = trackId;
        command.Ip = "127.0.0.1";
        command.Tags = ServiceLocator.Current.GetInstance<IStatisticTagsHelper>().GetTags(false); // don't get all tags
        command.Hit.Position = null;
    });
}
```

Here is a compare image of the tags that get sendt after and before I have added false to the function
![find tags compare]({{ site.baseUrl }}/images/2017-10-05/tags_compare.png)