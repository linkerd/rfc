- Contribution Name: ci-health-metrics
- Implementation Owner: @alpeb, @kleimkuhler
- Start Date: 2020-03-16
- Target Date: 2020-04-03
- RFC PR: [linkerd/rfc#11](https://github.com/linkerd/rfc/pull/11)
- Linkerd Issue:
  [linkerd/linkerd2#4176](https://github.com/linkerd/linkerd2/issues/4176)
- Reviewers: @grampelberg

# Summary

[summary]: #summary

This RFC proposes a service that would monitor the results from each CI run. It
should allow us to catch consistently flaky tests.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

CI jobs inevitably fail occasionally, either because of real problems with the
code, or because of test flakiness. We'd like to stay informed about what tests
are being persistently flaky, without having to dig into the CI run results
themselves. This will allow us to properly prioritize work to fix those
failures.

More specifically we'd like to get, for a given time span, the list of error
messages with their frequency aggregated globally, per workflow and per job.

The first problem to address will be to expose the CI jobs results, which
currently are only available through Github's UI. With access to that data in
the appropriate format, we'll then be able to determine how to persist and
present it to operators.

# Design proposal (Step 2)

## Exposing the errors

There are two ways to expose CI run failures: annotations and artifacts.

### Annotations

Annotations are created whenever an error message from any of the CI steps is
sent to stdout/stderr using the appropriate format, described
[here](https://help.github.com/en/actions/reference/workflow-commands-for-github-actions#setting-an-error-message).
Annotation messages are visible in the summary page of a CI run. Currently our
error messages aren't following that format, and the summary page only shows
something like "Process completed with exit code 1." whenever a job fails.
Annotations are surfaced through Github's Checks API.

### Artifacts

Instead of conforming to Github's annotation format, we could use a more
standard format such as xUnit XML reports. That would imply building some
wrapper around the tests to accommodate that format, and the results would be
then uploaded to the CI run's artifact repository (which we're already
leveraging to store docker images). Then another tool can consume those reports.

But I haven't found such a tool that would consume xUnit reports for showing
historical trends, which is the main goal of this RFC (being able to catch
consistently flaky tests). For this reason I'm proposing we go down the
annotations route.

(Although do raise your voice if you find an appropriate tool for xUnit
reports!)

## Formatting

We have integration tests and checks written in Go, Javascript and Bash. Each of
those environments would require changing how they report errors to properly
format them as explained above.

For example Go test failures would need to be modified like so:

```
go fail(t, "linkerd upgrade config command failed", "linkerd upgrade config command failed\n%s\n%s", out, stderr)
```

Where `fail` would output a string like

```
::error file=upgrade_test.go,line=103::linkerd upgrade config command failed
```
that is interpreted by Github as an annotation. That function would also make
the original `t.Fail("linkerd upgrade config command failed\n%s\n%s", out,
stderr)` call to report the to Go's testing framework.

Note that the annotation will just contain the generic error message (no
details) as we'd like to aggregate the number of appearances of such messages.

## Available Data

Once the error information is exposed, these are the pieces of data available
for each CI run:

- Check Suite ID
- Check Run ID
- Workflow name
- Job name
- Completion status 
- Start timestamp
- Finish timestamp
- Test/Check name
- File name
- Error message

Which are retrieved using the following Github API calls:

```
# This gives us a list of check suite IDs:
GET /repos/linkerd/linkerd2/actions/runs

# For each check suite ID, this gives us the check run ID, job name, workflow
# ID, completion status and timestamps:
GET /repos/linkerd/linkerd2/check-suites/:check_suite_id/check-runs

# For each workflow ID, this gives us the workflow name
# (this can be cached across check runs):
GET https://api.github.com/repos/linkerd/linkerd2/actions/workflows/:workflow_id

# For each check run ID, we invoke the annotations API which gives us the file
# name and error message
GET repos/linkerd/linkerd2/check-runs/:check_run_id/annotations
```

## New Workflow for Reports

After each CI run finishes, a new Github Actions event would be dispatched that
should be caught by a new, separate workflow whose sole responsibility is to
build a report.

That workflow will first fetch the data described above. While data is available
for approximately 3 months back, we're only interested in the data for the last
  month. We have the option to fetch and process all the data upon each workflow
  run, which will alleviate us from dealing with any kind of state.
  Alternatively, we could only fetch data since the last time the workflow run,
  add the data to some cloud DB service, and then query that store to retrieve
  last month's data. As a first approach, and given the small amount of data we
  have to deal with, let's fetch all the data each time see how that performs.

## Reports

The reports would consist in tables and their corresponding charts for the
following data:

- Success rate
- Error frequency (showing most frequent error messages)
- Duration

We can have one set of tables/charts for last month's data and another for the
last week's data. Also, we can aggregate per workflow and per job, in the same
chart.

The final report can be an HTML page containing the tables and pointing to chart
files generated  with a library like [chart](https://github.com/vdobler/chart),
all stored as workflow artifacts.

# Prior art

[prior-art]: #prior-art

Github's Checks API have only until recently become beta, so it's
expected there aren't well known tools that leverage that information.

I did find the [BuildPulse](https://github.com/marketplace/buildpulse/) Github
App, that specializes on tracking flaky tests in Github Actions. I couldn't make
it work in a Linkerd2 fork. And presumably it will also require the changes
described in Phase 1 above for it to properly give us actionable data instead of
data about the generic error messages we currently have. BuildPulse is closed
source and a paid service.

# Future possibilities

[future-possibilities]: #future-possibilities

Once we expose the CI metrics, we open the door to present this data however we
wish in the future.

Reports will be stored as artifacts for the new workflow. For ease of
consumption, we can think of publishing them under some section of the website.
