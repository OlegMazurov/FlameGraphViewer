# Advanced Flame Graph Viewer

A standalone GUI application to provide enhanced uniform analysis of performance data from various sources.

## Features
* Drag-and-drop/open one or multiple data files to get an aggregated view 
* Supported formats:
  * original Flame Graph's *\*.svg* and *\*.json*
  * Java Flight Recorder's *\*.jfr*
  * Oracle Studio collector's *\*.er*
  * *async-profiler*'s *\*.svg* and *\*.html*
* Switch between multiple metrics (f.e. Wall clock time and CPU time) where available
* Aggregate stack traces at their roots (*callees*) or leaves (*callers*)
* Aggregate over a single method or an arbitrary stack trace fragment 
* Flat view with exclusive (*self*) and inclusive (*cumulative*) metrics
* Color a single method or a selected set of methods
* Filter in or out methods, stack trace fragments, threads, files
* Move back and forth in history
* Export the current view as a self-contained *flamegraph.svg*

## Rationale 

The Flame Graph visualization tool, introduced and popularized by [Brendan Gregg](https://www.brendangregg.com/flamegraphs.html), 
has become ubiquitous in analysis of stack trace based performance data.
Providing a compelling presentation and convenient navigation, the original implementation, however, didn't dare to go beyond 
Call Tree views, already implemented by many performance tools at the time, functionally: there was no new question that 
could be answered with Flame Graph and that would be out of reach for Call Tree views. 
That was especially frustrating as the *gprof* approach to aggregating performance metrics for functions along with their 
callers and callees, or parent and children in the call tree hierarchy, had been well known and readily available for the same 
visualization. 
Moreover, extending the callers-callees aggregations to [stack trace fragments](https://docs.oracle.com/cd/E77782_01/html/E77798/afagg.html#OSSPAafagj), 
as was implemented in [Sun Studio Performance Analyzer](https://en.wikipedia.org/wiki/Performance_Analyzer), fit naturally
both presentation and navigation provided by Flame Graph but that apparently was missed by all involved parties.

This tool is intended to fill the gap. 

<...>

## Functionality

The basic functionality of the original Flame Graph implementation is mostly preserved.

The *Up/Down* arrow in the *Flame* view switches between aggregating stack traces at their roots and their leaves.
Another interpretation, which can naturally be extended to call trace fragments, is that the bottom-up aggregation
shows everything called from the currently selected fragment and the top-down aggregation shows everything from where
the currently selected fragment is called. When nothing is selected, all traces are shown.

A mouse click on a visible method extends or shrinks the currently selected stack trace fragment to that method.
Although it's possible to select a single method with a series of mouse clicks and the *Up/Down* button, a shortcut is 
to click on a method of interest while holding the *Shift* key on the keyboard.

The *Left* and *Right* arrows in the *Flame* view allow to easily go back and forth in the history of clicks.

<...>

### Exclusive and Inclusive metrics
The *Flat* tab provides a list of methods/functions with their *exclusive* and *inclusive* metrics.
The *exclusive*, or *self*, metric for method *m* is computed over all events for which the current, or stack trace *leaf*,
method was *m*. During parallel execution, *exclusive* metrics are added together for all contributing threads or processes paying no
attention to their possible overlaps in real time. That makes the metric stable: same workload should produce same metric values 
no matter whether its execution is parallel or serial (that is at least the goal, the reality is, of course, messier than that).

The *inclusive* metric for method *m* is computed over all events for which *m* is part of event's stack trace.
If I were to trace method *m*'s execution, the *inclusive* metric would match the accumulated difference of time, or whatever 
one measures, between method's entry and exit events. Parallel execution gets the same treatment as above. A special consideration 
should be given, however, to recursive invocation of *m*. To avoid overcounting, only the first occurrence of *m* in any 
stack trace should be counted towards its *inclusive* metric. (See more on *Recursion* below).

The *Flat* tab is synchronized with the *Flame* tab: method selection in one is preserved in the other. 

### Recursion
Why, when computing the *inclusive* metric of a recursive method *m*, do I count only the first occurrence of the method in any
stack trace? A simple argument can be derived from a general principle of metric stability: just reorganizing how some
amount of work is done without optimizing it should not change a measure of that work represented by a metric.
In a simple case of tail-call recursion, the finishing call can be more or less easily refactored into a loop. 
The amount of work, neglecting the difference between method call and loop implementation overheads, remains the same
and thus the metric measuring method execution should remain the same.

In a more complicated case of, say, measuring *parseExpression()* method execution, the same method may be called
recursively many times in various contexts. Yet, I would argue, the same logic applies. Even if flattening out  
*parseExpression()* 's execution would not be an easy task, though however impractical not impossible, it justifies 
not counting methods occurrences after the first one. From a different angle, one would be surprised if the *inclusive*
metric of *parseExpression()* exceeded, or even greatly exceeded, that of, say, *parseProgram()* when it is known that
the former is only called from the latter, which is invoked only once. The described approach avoids any such surprises.

How to extend this logic to computing metrics for callers and callees of a recursive method *m*, or any recursive stack
trace fragment? Note, that things may become really convoluted in the latter case. Consider multiple occurrences of 
fragment *(a -> b -> a)* in stack trace *(main -> a -> b -> a -> b -> a -> c)*. Should this trace's metric be added to
caller *main* or *b*? Callee *b* or *c*? Should those two attributions be correlated (f.e., if *main* is the caller then 
*b* is the callee)?

First, let's introduce the following invariant. For any non-recursive non-root method, the sum of all metrics attributed 
to its callers must be equal to the sum of all metrics attributed to its callees plus method's *exclusive* metric and that
must be equal to method's *inclusive* metric. Why so? Whenever we encounter a method in a stack trace it's always called
from some caller and either calls one of its callees or is the leaf method. A simple physical law of conservation: whatever
flows in must either flow out or stay inside.

Preserving that invariant in the case of recursion leads to the following approach of attributing metrics to callers and 
callees of recursive methods or trace fragments. In any stack trace, pick the caller of the first occurrence of the fragment
and the callee, if any, of its last occurrence. In the example above, *main* will be picked up as the caller and *c* as 
the callee, thus there will be no correlation between the choices. 

If a single recursive method is selected in the *Flame* view, it may appear that its recursive nature is totally
omitted from the presentation whether the bottom-up (*callees*) or the top-down (*callers) view is selected. It may be 
easily revealed, however, by adding a caller or a callee to the selected fragment and switching to the opposite view.
Any existing stack trace fragment can be selected and analyzed with a series of method selections and switching between 
the callers and callees views, f.e. *(a -> a -> a)* or *(a -> b -> a -> b)*.

### Colors

Colors are helpful in highlighting or separating methods or subsets of methods demonstrating a particular performance issue.
Default colors are pseudo-randomly picked from pre-defined palettes based on method names. Those colors are stable 
between multiple invocations of the Flame Graph Viewer (though may change between different versions of the viewer).

For a subset of methods, the selected color becomes the base of the new palette from which all other colors are randomly
picked. 

<...>

### Filters
You can filter in or filter out all events that map to the selected set of objects. 
Additional filters may be available in context menus.
Upon applying a filter, all tabs are updated to show their aggregated views of the resulting subset of events. 
Consecutively applied filters are conjunctive (their predicates chained by AND).

<...>

### Export

You may have done a good deal of analysis of a huge amount of data coming from several files and, with the use of 
filters and colors, get a nice presentation of a particular performance issue you'd like to share with your colleagues.
Taking a snapshot of the *Flame* tab view is a straightforward approach. A better one is taking a "snapshot" of the 
current view by *exporting* it to an *\*.svg* or *\*.json* file.
The SVG file is a self-contained original flame graph representation ready for interactive work in a browser.
The JSON file only represents the data model of the flame graph.

Attach exported files to bug reports, email them to a coworker, feed them to other processing tools.

<...>
