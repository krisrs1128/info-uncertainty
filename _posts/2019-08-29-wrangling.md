---
title: "'What is a sample, even?' and the Existential Questions of Data Wrangling"
date: 2019-08-29 15:35 -4
---

### TODO: Include code examples

In your life as a data scientist, you will spend a lot of time trying to
generate new artifacts from "raw" data. These artifacts are tangible things,
living on a device, a website, and yes, maybe even on paper, including things
like,

* *Model predictions*, which are usually a component of some (potentially
   automated) decision making process. Some examples are predictions about which
   song to recommend someone (automated) or which educational interventions to
   give to struggling students (not automated).
* *Model summaries*, which inform our understanding of the data generating's
   systems underlying mechanics. Some examples might be quantitative measures
   about whether living in Montreal influences your inherent interest in heavy
   coats or whether smoking causes lung cancer.
* *Figures*, which communicate salient patterns in a dataset, to inform
   understanding or decision-making[^1]. For example, should you [invade
   Russia](https://en.wikipedia.org/wiki/File:Minard.png)?
* *Reports*, which synthesize of a few parts of a data analysis to provide
   evidence for some higher-level theory.

No matter what your downstream artifact will be, you'll need to do a bit of work
upstream, mainly around,

1. Problem Formulation: What are you trying to do, and why does it matter?
2. Data Wrangling: How does the data need to be stretched, squeezed, and srubbed
   to make it suitable for automatic, algorithmic processing.
   
From the previous lecture / rant, we know that data are diverse, so it's no
surprise that the answers to these questions will be very context dependent.
That said, I find a few general principles relevant in almost any application.
The principles for (1) are profound, we'll hopefully revisit them later. But (2)
is within reach even at this point, so let's focus on that.

In the abstract, we'd love to think of our data as a big matrix $$X$$ with $$n$$
rows and $$p$$ columns. The rows correspond to samples and the columns
correspond to attributes. The $$ij^{th}$$ element of this matrix will be
$$x_{ij}$$, we'll often write the $$i^{th}$$ row as just $$x_i$$.

For example, each row might be a person in this class. The first column might be
name, the second might be major, and the third might be favorite pokemon. Or,
the samples might be photos of handwritten digits, and the columns might measure
intensity at different [pixels](https://observablehq.com/@mbostock/lets-try-t-sne)[^2].

Unfortunately, there are all sorts of obstacles that keep us from building this
beautifully mathematical $$X$$. Let's consider them one at a time, and what you
might be able to do about it.

## I have no idea what these columns are

This isn't strictly a barrier to putting everything into a single $$X$$, but if
you are confused by this, you should stop coding and write to the people who
shared the data with you. You don't want to try doing anything to the data if
you're not even sure what they are.

## Managing column types

Different columns will have different types, for example,

* Numeric
* (Ordered) categorical 
* Dates
* (Positive) Integer
* Text

and you want to make sure that the types stored in the computer reflect your
conceptual understanding of what that column should contain. For an example of
the type of conceptual mismatch that could cause trouble later on, the computer
might accidentally read in a numeric column as a string, because some entries
were sloppily recorded as strings -- looking at the raw input file might reveal 
that the "height" variable has entries like `(..., 10.1, 9.2, forgot measuring
stick, 10.9, ...)`. In this case, you'll want to convert the string to a missing
value and coerce the column to a numeric.

Dates can be annoying, but a bit of fiddling with
[strftime](http://strftime.net/) will usually get you what you need. Depending
on the task, it can be helpful to isolate some features from the dates -- was
the date on a weekend, a holiday, or during rush hour?

Categorical variables deserve some additional commentary, since they can be
especially tricky. First off all, you might notice that different categories are
actually referring to the same thing. One of the categories might be `POTATO`,
while another reads `potato`. Unless you go in and consolidate these, they will
be treated as different categories in downstream analysis.

Second, there are often many very rarely seen categories. It often makes sense
to lump these all into an "other" category.

It may also be useful to split a categorical variable that encodes several pieces
of information into different columns.

You might want to rename the categories (and sometimes column names) for the
sake of readability -- really long column names can be a pain. That said, if
it's going to take forever to do more than a few automatic conversions (e.g.,
replacing spaces, converting to lower case), it may not be worth the time.

Finally, you may want to k-hot encode your factors. This is usually not needed
for visualization, but ML algorithms often need $$X$$ to be a completely numeric
matrix.

## Missing values

You want to make sure that missing values are properly documented. This means
replacing things like "" and -999999 with the appropriate NA.

It is sometimes also a good idea to distinguish between structurally and
randomly missing values. Structurally missing values are entries that are not
recorded for some reason that applies equally to many entries, e.g., a question
might not have been asked on a survey in one of the years. Randomly missing
values are isolated missing values, whose reasons for being missing are not
clearly characterized.

## Joining and Reorganizing

Often, the thing you want to think about as a unified $$X$$ lives in a tangled
web of inconsistent files, in subdirectories that seem to have been nested for
no particular reason other than to torment the data scientist who was foolhardy
enough to think that the data could be properly analyzed. Actually, there are
often good reasons for this incoherence, including

* Databases: The data may have lived in a relational database, where rows in
  different tables corresponded to different conceptual units. Hence, you are
  given one table where each row describes a school, and another where each row
  is a student. If you want to see characteristics of a student's school when
  analyzing their test results, you're going to have to join these tables
  together somehow.
* Streaming-ness: The data may have been automatically generated by some
  platform. It might spit out a different record every minute, say. You'll want
  to combine these as rows $$x_i$$ of a newly formed matrix $$X$$. 

A few caveats are in order. There first is that the data may be of different
types, and it might not make sense to join them. For example, you may have a
collection of satellite images, along with metadata associated with them (date
taken, geographic coordinates, sensor used). In this case, I recommend putting
the "nontabular" signals (images, audio recordings, ...) into a directory and
including that path in a processed, metadata file.

A related caveat is that the data might be too big to merge into one file --
this is especially the case with streaming data or data that you only ever
intend to handle in small batches at a time. Nonetheless, you still need to do
this wrangling, if only to (a) ensure consistency across the pieces and (b)
build a metadata file linking the pieces.

The most profound exception to the recommendations above is that what we think
of as a single sample might vary from one analysis goal to another. Indeed, the
whole idea of a "sample size" is fraught, we've been lying to you all along so
that you don't have an existential crisis too early.

The issue often arises when data are not i.i.d. or have some sort of multiscale
structure. For example, in a longitudinal study, you might have tracked the same
few people for many years, recording information about them at particular
intervals. You have a choice here: each row $$x_i$$ might include all the data
you ever captured about person $$i$$. Alternatively, it might be reasonable to
split each person's data across several rows, each corresponding to a timepoint.

Another example is this bikesharing data. Maybe you should group the 24h periods
into rows, or maybe you should consider each timepoint separately.

There's no universal rule, what you do should depend on your analysis goal (and
potentially on the constraints imposed by your computing environment).


## Final Points

When you go through these wrangling steps, you'll be changing your original
data. For this reason, I urge you to always keep the raw data untouched, and
write the results of this wrangling to new processed files. These processed
files are what your algorithmic and inferential code will be touching
downstream.

While I've listed some of my best practices, all sorts of unexpected things can
happen. I've seen excel footnotes mess up numerical columns, tables which
almost, *but not quite* joined, and survey results from 2099.

So, while "Let $$X \in \mathbb{R}^{p}$$" might be the first line in your
statistics or ML textbook, just getting to this point can involve quite a bit of
work in any real-world data science problem. Don't be deterred though -- the
problems you can solve if you have the ability to sift through real-world data
are ultimately the most interesting and impactful ones.


[^1]: They might even guide whether to collect different types of data.
[^2]: See the section on "Images."
