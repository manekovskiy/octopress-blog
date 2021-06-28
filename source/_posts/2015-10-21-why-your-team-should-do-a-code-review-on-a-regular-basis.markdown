---
layout: post
title: "Why Your Team Should Do a Code Review on a Regular Basis"
date: 2015-10-21 19:38
comments: true
categories: Development
---

While working with any system we have to take into account so many aspects that even armed with the best tooling and extensive test suites we cannot guarantee 100% (but surely we can plan and minimize the risks) that the development/maintenance cost of our software will not exceed the amount of profit it generates.

And to minimize the risks and costs we have to follow the best practices, methodologies and techniques. My personal approach to aforementioned good practices is very pragmatic - if something proved that it could be used to simplify the life of the team we'll use it. And one of that things that are often underestimated or neglected is a code (or peer) review.

> In software development, peer review is a type of software review in which a work product (document, code, or other) is examined by its author and one or more colleagues, in order to evaluate its technical content and quality.
> 
> **From Wikipedia, the free encyclopedia.**

## What Are The Benefits Of Doing A Code Review?

The main intent of the code review is to identify source code defects and quality issues. Another big advantage is a knowledge transfer. This is maybe the least expected outcome of the code review process. I personally observed lots of cases when reviewers were giving links and citing some external resources in their comments. That additional resources were helping author to get deeper into details, see the issue from different angles and as a result produce better code. 

For those who like numbers I recommend to read a study by Bacchelli A. and Bird C. [Expectations, outcomes, and challenges of modern code review](http://sback.it/publications/icse2013.pdf) that characterizes the motivations of developers and managers for code review and compares it with actual results.

Also Steve McConnell gives enough facts of code review effectiveness in [Code Complete](http://www.amazon.com/gp/product/0735619670/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0735619670&linkCode=as2&tag=manekblog-20&linkId=63IP6PEWIO325YEU):

<blockquote style="font-style: normal">
	<p style="margin-left: -1em; margin-bottom: 0.5em;">Technical reviews have been studied much longer than pair programming, and their results, as described in case studies and elsewhere, have been impressive:</p>
	<ul>
		<li>IBM found that each hour of inspection prevented about 100 hours of related work (testing and defect correction) (Holland 1999).</li>
		<li>Raytheon reduced its cost of defect correction (rework) from about 40 percent of total project cost to about 20 percent through an initiative that focused on inspections (Haley 1996).</li>
		<li>Hewlett-Packard reported that its inspection program saved an estimated $21.5 million per year (Grady and Van Slack 1994).</li>
		<li>Imperial Chemical Industries found that the cost of maintaining a portfolio of about 400 programs was only about 10 percent as high as the cost of maintaining a similar set of programs that had not been inspected (Gilb and Graham 1993).</li>
		<li>A study of large programs found that each hour spent on inspections avoided an average of 33 hours of maintenance work and that inspections were up to 20 times more efficient than testing (Russell 1991).</li>
		<li>In a software-maintenance organization, 55 percent of one-line maintenance changes were in error before code reviews were introduced. After reviews were introduced, only 2 percent of the changes were in error (Freedman and Weinberg 1990). When all changes were considered, 95 percent were correct the first time after reviews were introduced. Before reviews were introduced, under 20 percent were correct the first time.</li>
		<li>A group of 11 programs were developed by the same group of people, and all were released to production. The first five were developed without reviews and averaged 4.5 errors per 100 lines of code. The other six were inspected and averaged only 0.82 errors per 100 lines of code. Reviews cut the errors by over 80 percent (Freedman and Weinberg 1990).</li>
		<li>Capers Jones reports that of all the software projects he has studied that have achieved 99 percent defect-removal rates or better, all have used formal inspections. Also, none of the projects that achieved less than 75 percent defectremoval efficiency used formal inspections (Jones 2000).</li>
	</ul>
</blockquote>

## How It Is Working?

The typical code review process is following:

{% img center /images/posts/code-review-process.png 'Diagram showing the typical code review flow' %}

 1. Author of change generates a patch and sends it to the code review system
 2. Author invites his teammates to review the code
 3. Code review participants are adding comments and suggestions on code improvement
 4. Author either follows the suggestions and updates the code or rejects them
 5. The code review is updated by author and a new review iteration is started
 6. When all debates around the change are finished the code review is approved and the change is merged into the repository.

## Where To Start?

Code review is often supported by tools preferably integrated into the development environment. If you are working alone there is a site where you can ask for a peer programmer code review - [Code Review](http://codereview.stackexchange.com/). As like on a Stack Overflow this site has an army of active members that would happily help you no matter what language or technology you are using.

There are also a plenty of tools available on a market starting from aforementioned SO site to [TFS support](https://www.visualstudio.com/en-us/get-started/code/get-code-reviewed-vs "Get your code reviewed with Visual Studio") and integrated code review tooling inside [GitHub](https://github.com/features#code-review "Collaborative code review on GitHub"), [GitLab](https://about.gitlab.com/features/ "GitLab features"), [Bitbucket](https://bitbucket.org/product/features "Bitbucket features") and other OSS collaboration platforms.

## Conclusions

If you are looking for how you can improve the state of the codebase and/or development process in general start practicing code review on daily basis.

Surely if the team never practiced code review before it would be harder to start but as Laozi said “The journey of a thousand miles begins with one step”. And I wish you to succeed!