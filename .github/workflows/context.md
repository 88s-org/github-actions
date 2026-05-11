The 9 Contexts
1. github — The most used context
Info about the event that triggered the workflow and the repo.

Property	Example Value	Jenkins Equivalent
github.ref_name	main	env.BRANCH_NAME
github.sha	a1b2c3d4...	env.GIT_COMMIT
github.actor	siva	currentBuild.rawBuild.getCause(...)
github.event_name	push	(no equivalent)
github.run_number	42	env.BUILD_NUMBER
github.repository	siva/myapp	env.JOB_NAME
github.workspace	/home/runner/work/myapp	env.WORKSPACE
github.event	full webhook payload (JSON object)	(no equivalent)

- run: echo "Triggered by ${{ github.actor }} on ${{ github.ref_name }}"
- run: echo "PR title: ${{ github.event.pull_request.title }}"  # only on PR events
2. env — Environment variables
Variables set with env: at workflow, job, or step level.


env:
  APP_NAME: myapp

jobs:
  build:
    steps:
      - run: echo ${{ env.APP_NAME }}    # via context
      - run: echo $APP_NAME              # via shell — both work, prefer shell for run:
Rule: In run: steps, use shell $VAR. Use ${{ env.VAR }} in places where the shell isn't available — like if:, with:, name:.

3. secrets — Secret values
Secrets stored in GitHub → Settings → Secrets and Variables.


- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}   # only way to access secrets

# One special secret that's always available (no setup needed):
- run: echo ${{ secrets.GITHUB_TOKEN }}           # auto-generated per run
Secrets cannot be used in if: conditions — GitHub blocks this intentionally.

4. vars — Configuration variables (non-secret)
Like secrets but visible in logs. Set in Settings → Secrets and Variables → Variables.


# Good for: URLs, feature flags, config values that aren't sensitive
- run: |
    echo "SonarQube URL : ${{ vars.SONAR_URL }}"
    echo "ECR Registry  : ${{ vars.ECR_REGISTRY }}"
    curl ${{ vars.SONAR_URL }}/api/health
vars vs env: vars is stored in GitHub settings (survives across runs, set by admins). env is defined in the YAML file itself.

5. inputs — Manual trigger inputs
Only available when triggered by workflow_dispatch or workflow_call.


on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [ staging, prod ]

jobs:
  deploy:
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
      - if: inputs.environment == 'prod'    # use in conditions
        run: echo "Extra prod checks..."
Jenkins equivalent: params.ENVIRONMENT from parameters { choice(...) }.

6. steps — Outputs from previous steps (same job)
Access the output or outcome of a step that ran earlier in the same job.


steps:
  - name: Compute tag
    id: tag                                    # step MUST have an id
    run: echo "value=myapp:abc123" >> $GITHUB_OUTPUT

  - name: Use that output
    run: echo "Image is ${{ steps.tag.outputs.value }}"

  - name: Check if previous step failed
    if: steps.tag.outcome == 'failure'
    run: echo "tag step failed"
steps.<id>.	Meaning
.outputs.<name>	value written to $GITHUB_OUTPUT
.outcome	success, failure, skipped, cancelled
.conclusion	same as outcome but accounts for continue-on-error
7. needs — Outputs from upstream jobs
Access outputs from jobs listed in needs:. This is how you pass data between jobs.


jobs:
  build:
    outputs:
      image-tag: ${{ steps.tag.outputs.value }}   # job-level output
    steps:
      - id: tag
        run: echo "value=abc123" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image-tag }}"
      - if: needs.build.result == 'success'        # check job result
        run: echo "build succeeded"
needs.<job>.	Meaning
.outputs.<name>	value from the job's outputs: block
.result	success, failure, skipped, cancelled
8. job — Current job info
Info about the currently running job.


- run: echo "Job status so far: ${{ job.status }}"
# Values: success, failure, cancelled
# Useful in if: conditions on cleanup steps
9. runner — The runner machine
Info about the machine running the job.


- run: |
    echo "OS    : ${{ runner.os }}"        # Linux, Windows, macOS
    echo "Arch  : ${{ runner.arch }}"      # X64, ARM64
    echo "Temp  : ${{ runner.temp }}"      # /tmp (safe scratch space)
    echo "Tool  : ${{ runner.tool_cache }}"# where setup-* actions install tools
Where Each Context is Available
Not every context works everywhere — this is a common source of bugs:

Context	on:	env:	jobs.<id>.if:	steps.if:	run:	with:
github	✅	✅	✅	✅	✅	✅
env	❌	✅	✅	✅	✅	✅
secrets	❌	❌	❌	❌	via env:	✅
vars	❌	✅	✅	✅	✅	✅
inputs	❌	✅	✅	✅	✅	✅
needs	❌	❌	✅	❌	❌	❌
steps	❌	❌	❌	✅	❌	✅
job	❌	❌	❌	✅	❌	❌
runner	❌	❌	❌	✅	✅	✅
Key gotcha for students: needs is only available at the job level if: — you can't use it inside a step's if:. Use needs outputs via env: to bring it into steps.

One Mental Model to Remember

github  → WHERE and WHY (repo, event, actor, sha)
env     → WHAT you configured in the YAML
vars    → WHAT admins configured in GitHub settings
secrets → CREDENTIALS (write-only, never readable back)
inputs  → WHAT the user typed when running manually
steps   → WHAT this job's steps produced
needs   → WHAT upstream jobs produced
job     → HOW this job is going right now
runner  → WHERE this job is running