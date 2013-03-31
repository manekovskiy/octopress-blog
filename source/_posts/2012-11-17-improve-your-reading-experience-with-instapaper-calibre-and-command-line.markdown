---
comments: true
date: 2012-11-17 13:39:17
layout: post
slug: improve-your-reading-experience-with-instapaper-calibre-and-command-line
title: Improve Your Reading Experience with Instapaper, Calibre and Command Line
wordpress_id: 207
categories:
- Lifehacking
tags:
- Calibre
- Instapaper
- Reading
---

After I read Scott Hanselman's post "[Instapaper delivered to your Kindle changes how you consume web content - Plus IFTTT, blogs and more](http://www.hanselman.com/blog/InstapaperDeliveredToYourKindleChangesHowYouConsumeWebContentPlusIFTTTBlogsAndMore.aspx)" I bethought that I wanted to create an automated [Instapaper](http://www.instapaper.com) to my e-book reader "contend delivery system". Now as I finished here is my little story.


{% img right /images/posts/lbookv5-ereader.jpg 'LBook V5' 'LBook V5' %}Almost a year ago when I started using [Instapaper](http://www.instapaper.com) I realized that it would be great to grab all articles that were collected through the week, convert them to [EPUB](http://en.wikipedia.org/wiki/EPUB) format and send electronic book to my e-book reader device. The only problem was in my device - [Lbook V5](http://lbook.ua/products/lbooks/v5/). Yes, it is totally outdated and old comparing to Kindle devices. It supports EPUB but does not have access to the Internet, so Instapaper _"download"_ feature doesn't work for me.

A few month ago I found [Calibre](http://calibre-ebook.com) - free and open source e-book library management application. It helped me to organize and manage all my electronic library and I'm totally happy with it. Calibre has everything that could be possibly needed - scheduler support, custom news source with interactive setup and converters to various e-book formats. But what most interesting and important Calibre has command line [_ebook-convert.exe_](http://manual.calibre-ebook.com/cli/ebook-convert.html) utility which could be driven by _recipe files_. [Recipes in Calibre](http://manual.calibre-ebook.com/news.html) are just Python scripts (with a bits of custom logic if it is needed to parse some specific news source).

Below is simple Calibre recipe:

{% codeblock lang:python %}
class AdvancedUserRecipe1352822143(BasicNewsRecipe):
	title          = u'Custom News Source'
	oldest_article = 7
	max_articles_per_feed = 100
	auto_cleanup = True

	feeds = [(u'The title of the feed', u'http://somesite.com/feed')]
{% endcodeblock %}

This defines RSS feed source at [http://somesite.com/feed](http://somesite.com/feed) and declares that there should be no more than 100 articles not older than 7 days. If we'll use it with _ebook-convert_ utility, it will automatically fetch news from specified feed and will generate e-book file. The command line to generate book is following:

{% codeblock lang:bat %}
ebook-convert.exe input_file output_file [options]
{% endcodeblock %}

When _input_file_ parameter is recipe _ebook-convert_ runs it and then produces e-book in specified by _output_file_ parameter format. Recipe should populate _feeds_ dictionary so _ebook-convert_ will know what XML feeds should be processed. Options could accept two parameters - _username_ and _password_ (correct me if I'm wrong but I didn't found any information about possibility to use other/custom parameters). That was a brief introduction to Calibre recipe files. Now here is the problem.

Calibre has built in [Instapaper recipe](http://khromov.wordpress.com/projects/instapaper-calibre-recipe/). This recipe was created by [Stanislav Khromov](http://khromov.wordpress.com) with [Jim Ramsay](https://bitbucket.org/lack). Recipe has two versions - stable (it is part of current Calibre release) and development version, both could be found on [BitBucket](https://bitbucket.org/khromov/calibre-instapaper).

The development version of Instapaper recipe does almost what I want, but I needed to extend its functionality including:

  * Grab articles from all pages inside one directory (yes, sometimes it happens, when I'm not reading Instapaper articles for a few weeks).
  * Merge articles from certain directories into one book.
  * Archive all items in directories. This actually implemented in development version, but instead of using _"Archive All..."_ form recipe emulates clicking on _"Move to Archive"_ button which takes a lot of time to process all items.


At first I decided to extend development version of the mentioned above recipe but after I wasted an hour trying to beat the Python I realized that I can write command line utility in .NET (where I feel myself very comfortable) which will do whatever I want and I will save a ton of time (I'm definitely not going to learn Python just to change/fix one Calibre recipe :)). So here is [InstaFeed](https://github.com/manekovskiy/InstaFeed) - little command line utility that can enumerate names of Instapaper directories, generate single RSS feed for specified list of directories and archive them all at once. It uses two awesome open-source projects - [Html Agility Pack](http://htmlagilitypack.codeplex.com/) and [Command Line Parser Library](http://commandline.codeplex.com/).


> Note: While this utility parses Instapaper HTML and produces RSS you can probably bypass "RSS limits" of Instapaper non-subscription accounts. But I encourage you to support this service. Cheating is not good at all, please respect [Marco Arment's](http://www.marco.org/) work and efforts he put in this awesome service.


Having the command line utility that produces locally stored RSS feeds the only thing that remains is to create simple Calibre Recipe for _ebook-convert_ utility. The recipe should be parameterized with path to RSS feed generated by _InstaFeed_. Here is the code:

{% codeblock lang:python %}
class LocalRssFeed(BasicNewsRecipe):
	title        = u'local_rss_feed'
	oldest_article    = 365
	max_articles_per_feed    = 100
	auto_cleanup    = True
	feeds = None

	def get_feeds(self):
		# little hack that allows passing path to local RSS feed as a parameter via command line
		self.feeds = [u'Instapaper Unread', 'file:///' + self.username]
		return self.feeds
{% endcodeblock %}

All custom recipes should be stored within _<Calibre Installation Directory>Calibre Settings\custom_recipes_ folder.


> Note: Everything in this post applies to Portable 0.8.65.0 version of Calibre for Microsoft Windows. I have no idea whether it will work for other versions or installation variants.


Below is sources for batch file that produces RSS feed from Read Later Instapaper directory and then generates e-book in EPUB format at _C:\Temp. _I run this batch weekly via Windows Task Scheduler.

{% codeblock lang:bat %}
@echo off
setlocal EnableDelayedExpansion
setlocal EnableExtensions

:: change path to your calibre and instafeed executables
set _instafeeddir=F:\util\instafeed\
set _calibredir=F:\util\Calibre Portable\

:: set output directory and naming convention here
set filename=C:\Temp\[%date:/=%]_instapaper_unread_articles
set rssfile=%filename%.xml
set ebookfile=%filename%.epub

%_instafeeddir%instafeed.exe -c rss -u <instapaper_username> -p <instapaper password> -d "Read Later" -o "%rssfile%"
%_calibredir%\Calibre\ebook-convert.exe "%_calibredir%\Calibre Settings\custom_recipes\local_rss_feed.recipe" "%ebookfile%" --username="%rssfile%"

endlocal
{% endcodeblock %}

I had fun writing [InstaFeed](https://github.com/manekovskiy/InstaFeed) and digging in Calibre recipes and hope that someone will benefit from my experience. What else could be said? Read with convenience and have fun!
