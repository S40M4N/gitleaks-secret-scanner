# Gitleaks lab results and walkthrough

Completed on 26 November 2026 in the GitHub web browser. All values used in this lab are published classroom examples or fictional test markers.

## Evidence to show the instructor

Open a run, select the **scan** job, and expand **Scan the selected repository with Gitleaks**. The failed runs below are intentional detections, not setup failures.

| Experiment | Run | Commit or target snapshot | Observed result |
| --- | --- | --- | --- |
| baseline | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738299846) | `38212f5c5bec037675cc43e0f8f123ee9cd7105f` | Pass: no leaks found |
| AWS example | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738357714) | `49bad4ce927461d6ba86e1046d9600eddfbf0152` | Fail: classroom-aws-example; config.py:2; 1 finding |
| AWS fix | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738467363) | `7a2665d168bfdca4191e38cb1d289e492337f687` | Pass: no leaks found |
| Demo before rule | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738516293) | `f8695fe87620d33774d2e759f09d46a691d74ccf` | Pass: no leaks found; LAB-DEMO-314159 at config.py:5 |
| Demo after rule | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738598197) | `1965e7f17216a0b7e282f07b9bd8ca1d4f668661` | Fail: classroom-demo-code; config.py:5; 1 finding |
| Six-case custom rule | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738759755) | `173b53b050235afc47435c831c47ebb28a55352b` | Fail: exactly 3 sakib-fictional-service-key findings at config.py lines 5, 6, 7; no findings at 8, 9, 10 |
| Cleanup | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738832955) | `9167014ff37e955e35e650272c0050fc040dc96f` | Pass: no leaks found; custom rule retained |
| tests-before-fix | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738863942) | `a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b` | Fail: generic-api-key; service_settings.py:4; entropy 5.000000; 1 finding |
| tests-after-fix | [Open scan](https://github.com/S40M4N/gitleaks-secret-scanner/actions/runs/45738895812) | `65706937738a63cbe9e9ea520c19ca30da069ab6` | Pass: no leaks found |

## What GitHub Actions runs

The workflow starts after a push or pull request, or through a manual workflow dispatch. It runs on an Ubuntu runner, checks out this fork, and installs Gitleaks **8.30.0** from its official release after checking the archive SHA-256. For the two external targets it also checks out a pinned commit of RafikHamza/tests.

The scanner prints the target and its full snapshot SHA, exports the selected commit with git archive, and runs Gitleaks dir on those exported files using this fork's .gitleaks.toml. Output is verbose and values are redacted. A finding makes the scanner return exit code 1 and the job fail. A clean scan prints **no leaks found**. This workflow scans a selected snapshot, not the full Git history. It runs after a push, so it does not prevent that push from exposing a real credential.

## AWS classroom example and fix

The published example was committed in config.py. Run #2 detected classroom-aws-example at line 2. Run #3 passed after replacing the literal with an environment lookup:

~~~python
import os

AWS_ACCESS_KEY_ID = os.environ.get("AWS_ACCESS_KEY_ID")
~~~

The application obtains its value from the runtime environment. The credential value is no longer embedded in the current source. A pattern finding does not prove a key is valid, and a passing scan does not prove an application has a configured runtime key.

The [earlier commit diff](https://github.com/S40M4N/gitleaks-secret-scanner/commit/49bad4ce927461d6ba86e1046d9600eddfbf0152) still shows the published AWS example. A new commit records a new snapshot; it does not edit earlier commits. That is why the latest scan passes while the older version still contains the example.

## Worked-example comparison

Run #4 contains the LAB-DEMO marker in config.py at line 5 but no matching classroom-demo-code rule, so it passes. Run #5 keeps the same config.py and adds the detection rule, so it fails with exactly one classroom-demo-code finding at line 5. The rule configuration changed between the two runs; the test marker did not.

## My fictional-service rule

The service prefix is SAKIB-KEY- and its format requires exactly eight digits. The original AWS course rule and the built-in rules are preserved. The final added rule is:

~~~toml
[[rules]]
id = "sakib-fictional-service-key"
description = "Fictional Sakib service key for classroom testing"
path = '''(^|/)config\.py$'''
regex = '''\bSAKIB-KEY-[0-9]{8}\b'''
~~~

The path limits this rule to config.py. The character class matches digits, {8} requires eight of them, and the word boundaries prevent matching an eight-digit substring of a longer digit sequence. This fictional format is only a classroom exercise; it is not evidence of an actual service credential.

### Six-case results table

All six values were committed together as Python comments in [test snapshot 173b53b](https://github.com/S40M4N/gitleaks-secret-scanner/blob/173b53b050235afc47435c831c47ebb28a55352b/config.py). Run #7 returned exactly three findings for sakib-fictional-service-key.

| Value | config.py line | Expected finding | Observed result |
| --- | --- | --- | --- |
| SAKIB-KEY-12345678 | 5 | Yes | Found at line 5 |
| SAKIB-KEY-87654321 | 6 | Yes | Found at line 6 |
| SAKIB-KEY-24681357 | 7 | Yes | Found at line 7 |
| SAKIB-CODE-12345678 | 8 | No: wrong prefix | No finding |
| SAKIB-KEY-1234567 | 9 | No: seven digits | No finding |
| SAKIB-KEY-123456789 | 10 | No: nine digits | No finding |

The line numbers in the redacted log identify which values matched. All six comments were removed afterward, the custom rule was retained, and run #8 passed with no leaks found.

## External project: why the built-in rule found a key

The before-fix scan printed snapshot **a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b**, matching the required target. It found **generic-api-key** in **service_settings.py, line 4**, with entropy **5.000000** and exactly one finding.

The file assigns a long, varied, random-looking literal to SERVICE_API_KEY. The API-key assignment provides credential context and the entropy value describes character variation. Together these make the synthetic value look like a hardcoded credential. This is a heuristic finding, not a check that the value can authenticate to a service.

The classroom rules do not apply because their paths are restricted to config.py. The configuration has [extend] useDefault = true, which keeps Gitleaks' built-in rules enabled, including generic-api-key.

In the same before-fix snapshot:

- [.env.example](https://github.com/RafikHamza/tests/blob/a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b/.env.example) contains the empty placeholder SERVICE_API_KEY= and the fictional endpoint.
- [.gitignore](https://github.com/RafikHamza/tests/blob/a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b/.gitignore) includes .env, keeping an untracked local environment file out of ordinary commits. It does not remove a literal already committed in Python source or erase history.
- The [README](https://github.com/RafikHamza/tests/blob/a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b/README.md) describes environment-based configuration, but service_settings.py line 4 actually assigns a literal. The README also acknowledges a deliberately hardcoded synthetic value in the Git history. Source inspection and the scan expose the mismatch in that snapshot.

## How I verified the external fix

The [fix commit](https://github.com/RafikHamza/tests/commit/65706937738a63cbe9e9ea520c19ca30da069ab6) changes only service_settings.py line 4: the hardcoded literal is replaced with:

~~~python
SERVICE_API_KEY = os.environ.get("SERVICE_API_KEY")
~~~

The after-fix scan printed snapshot **65706937738a63cbe9e9ea520c19ca30da069ab6**, matching the required fixed version, and reported **no leaks found**. The hardcoded value was removed from that source snapshot; the environment variable name is not itself a secret value. I reopened the [before-fix source](https://github.com/RafikHamza/tests/blob/a1bb4fccd5ba3a5c2ba07e76adbc9a8ae031a27b/service_settings.py#L4) after reviewing the fix and confirmed the older snapshot still contains the synthetic literal.

## If a real credential were exposed

First, promptly revoke or rotate the exposed credential at its provider so it can no longer grant access. Then update the application with the replacement through an appropriate secret store or runtime environment, remove the hardcoded value, investigate possible misuse, and coordinate any necessary history cleanup. Deleting a line or rewriting Git history alone does not invalidate a leaked credential. Pre-commit scanning can flag values before a commit; GitHub push protection can block supported secret patterns before a push completes.

Reference: [GitHub leaked-secret response guide](https://docs.github.com/en/code-security/tutorials/remediate-leaked-secrets/remediating-a-leaked-secret).

## Short live walkthrough

1. Show runs #4 and #5: the same marker passes before its rule and is found after its rule is added.
2. Show my final rule, run #7, and the six-case table: exactly the three valid-format comments matched.
3. Show run #8: test comments removed, rule retained, clean source passes.
4. Show run #9: correct before-fix SHA, generic-api-key, line 4, entropy 5.000000.
5. Show the one-line external fix and run #10: correct after-fix SHA and no leaks found.
6. Open the old source snapshots and explain why they still contain the examples; explain revocation as the first response to a real exposure.

AI assisted with carrying out this lab and drafting these notes. Reported outcomes were checked against the actual GitHub Actions logs and source snapshots. The live explanation remains the student's part of the assignment.
