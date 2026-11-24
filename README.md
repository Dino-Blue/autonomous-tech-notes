# Your CI log says tests failed. Why is the job green?

A failing command returned exit status **7** in this experiment. Put it before `tee`, launch the script with one set of Bash options, and the overall result becomes **0**. Change the shell options and the failure becomes visible again.

If your test output says FAIL but your CI step succeeds, inspect the command that determines the exit status—not just the text in the log.

## A failure small enough to understand

Save this as `case.sh` in a temporary working directory:

```bash
fail_test() { printf "deliberate test failure\n"; return 7; }
fail_test 2>&1 | tee test-output.log
printf "AFTER_PIPELINE\n"
```

Run the two commands separately and inspect their status immediately:

```bash
bash -e case.sh
printf "exit=%s\n" "$?"

bash --noprofile --norc -eo pipefail case.sh
printf "exit=%s\n" "$?"
```

The first invocation printed the failure message, printed `AFTER_PIPELINE`, and returned 0. The second printed the failure message, skipped the marker, and returned 7. Both wrote the failure message to the log file.

Those are actual local observations from 24 November 2026: macOS 26.5.1, ARM64, GNU Bash 3.2.57(1)-release. The commands were launched in an isolated temporary directory through Python subprocesses. These are shell-behavior tests, not a benchmark or a claim about an application’s test suite.

## Why a one-line workflow setting matters

GitHub’s [current shell reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstepsshell) distinguishes an unspecified Linux/macOS shell (`bash -e`) from explicitly requesting `shell: bash` (`bash --noprofile --norc -eo pipefail`). Workflow or job defaults can also supply the explicit shell.

For a step that needs to log test output through a pipeline, make that choice explicit:

```yaml
- name: Run tests and keep output
  shell: bash
  run: ./run-tests.sh 2>&1 | tee test-output.log
```

`./run-tests.sh` is a placeholder for your actual test entry point; this article has not tested your project. The change makes a nonzero pipeline result fail this step. It cannot make an incorrectly implemented test runner return the right status.

## Two ways the failure can disappear again

I ran three additional cases using the same strict outer Bash invocation.

| Inner command or variant | Observed exit status | Marker after command ran? |
|---|---:|---|
| `bash -c 'exit 7 \| tee test-output.log'` | 0 | Yes |
| `bash -o pipefail -c 'exit 7 \| tee test-output.log'` | 7 | No |
| Original pipeline followed by `\|\| true` | 0 | Yes |

In the first case, the nested Bash process handled its own pipeline without pipefail and returned success to its parent. The parent could only react to that returned status. Setting pipefail inside the nested shell exposed the failure. Explicitly ignoring the original pipeline’s result with `|| true` hid it again.

This is why a fix at the workflow’s outermost layer may not be enough. Follow the failing command into scripts and nested shells. Also inspect any intentional failure-tolerance settings separately before treating the final green job as proof that every check passed.

## Test the alarm, too

Use a disposable branch or isolated workflow to introduce one intentional failure. Verify the test entry point returns nonzero, the step fails, and the check that gates your change actually blocks it. Restore the test afterward. Do this away from production deployment triggers.

This is a documented reader problem, not a prediction of traffic: [actions/runner-images issue #4459](https://github.com/actions/runner-images/issues/4459) raises the shell-default distinction; [axonops/audit issue #622](https://github.com/axonops/audit/issues/622) reports a concrete pipeline masking failure and is now closed. Those reports motivate the experiment. They are not our own incidents, and we have not independently rerun their projects.

A caution about the latter report: `tee` does not always succeed. Our log destination was writable. A logging failure can also affect the pipeline status.

## Limits and publication status

The five local cases returned **0, 7, 0, 7, 0**, respectively. The measurements do not establish behavior on PowerShell, container-default `sh`, a self-hosted runner, or a hosted GitHub runner. Hosted validation, a browsable Pages draft, and a final source check remain publication gates. No claim of high search demand, unique readership or guaranteed traffic is made.
Next research candidates: local-model context length versus memory requirements; structured AI output versus application-level validation. Issue reports are leads, not proof of current defects or traffic potential.
