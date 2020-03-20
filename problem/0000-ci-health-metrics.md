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
should allow us to catch consistently flaky tests and have an overview of the CI
in terms of success rates and time performance.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

CI jobs inevitably fail occasionally, either because of real problems with the
code, or because of test flakiness. We'd like to stay informed about what tests
are being persistently flaky, without having to dig into the CI run results
themselves. This will allow us to properly prioritize work to fix those
failures.

Besides identifying the most common failures, by surfacing error metrics from the
CI jobs we'll also have at our disposal an overview of CI's performance, in
terms of things like success rates and time taken by the jobs. This information
can be aggregated at the global, workflow and job levels.

The first problem to address will be to expose the CI jobs results, which
currently are only available through Github's UI. With access to that data in
the appropriate format, we'll then be able to determine how to persist and
present it to operators.

# Design proposal (Step 2)

Will be detailed once Step 1 gets approved.

# Prior art

[prior-art]: #prior-art

Github's Workflows and Checks API have only until recently become beta, so it's
expected there aren't well known tools that leverage that information.

I did find the [BuildPulse](https://github.com/marketplace/buildpulse/) Github
App, that specializes on tracking flaky tests in Github Actions. I couldn't make
it work in a Linkerd2 fork. And presumably it will also require the changes
described in Phase 1 above for it to properly give us actionable data instead of
data about the generic error messages we currently have. BuildPulse is closed
source and a paid service.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Hosting of the Prometheus exporter and Grafana dashboard that this solution
would require. Presumably AKS?

# Future possibilities

[future-possibilities]: #future-possibilities

Once we expose the CI metrics, we open the door to present this data however we
wish in the future.
