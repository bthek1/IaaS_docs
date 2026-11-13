# Pulumi

**Short version:** Pulumi is Terraform’s model — declarative resource graph, provider plugins, state file, preview/apply — but you write the program in Python/TypeScript/Go/C#/Java instead of HCL. Its two genuine differentiators are the Automation API (drive Pulumi as a library from your own app) and real testability. Everything else is roughly at parity with Terraform/OpenTofu, with a smaller ecosystem.

Author

Benedict Thekkel

## Core model

You write a program; running it registers resources with the engine over gRPC. The engine diffs the desired graph against state and calls providers.

``` python
import pulumi
import pulumi_aws as aws

bucket = aws.s3.BucketV2("assets")
aws.s3.BucketVersioningV2("assets-versioning",
    bucket=bucket.id,
    versioning_configuration={"status": "Enabled"})

pulumi.export("bucket_name", bucket.id)
```

Most providers are auto-bridged from the Terraform providers, so the resource surface and schema are near-identical — including the warts. Error messages sometimes leak HCL-isms.

**Stacks** are environments (`dev`, `prod`), each with its own state and config file (`Pulumi.dev.yaml`). `pulumi config set --secret db:password` encrypts per-stack.

------------------------------------------------------------------------

## The one thing that will trip you up: Output\[T\]

Resource properties are `Output[T]` — a future that also carries dependency edges and a secret flag. You can’t branch on them or f-string them:

``` python
# wrong — Output isn't a str
url = f"https://{bucket.bucket_domain_name}"

# right
url = pulumi.Output.concat("https://", bucket.bucket_domain_name)

# multiple inputs
conn = pulumi.Output.all(db.address, db.port).apply(
    lambda a: f"postgres://{a[0]}:{a[1]}/app")
```

During `preview`, unresolved values are `unknown`, so anything computed inside `.apply()` is invisible in the plan. Avoid creating resources inside `apply` — you lose preview fidelity and dependency clarity. This is the single biggest source of Pulumi frustration.

------------------------------------------------------------------------

## What’s actually better than Terraform

**Automation API.** Embed the engine as a library and run stacks programmatically — no CLI, no shell-out. This is the killer feature for a SaaS: per-tenant infrastructure provisioned from a Django management command or Celery task, with real error handling.

``` python
from pulumi import automation as auto

stack = auto.create_or_select_stack(
    stack_name=f"tenant-{clinic_id}",
    project_name="tenant-infra",
    program=lambda: define_tenant(clinic_id))
up = stack.up(on_output=print)
```

**Testing.** Mock the provider layer and unit-test your infra logic:

``` python
pulumi.runtime.set_mocks(MyMocks())
# then assert on tags, naming, policy compliance
```

Terraform has nothing comparable without `terraform plan` round-trips.

**Components.** `ComponentResource` subclasses are actual classes — constructors, inheritance, typed args. Substantially better than TF modules for abstraction. You can package a component and consume it from another language.

**Loops/conditionals.** No `count`/`for_each` gymnastics. Just write the loop.

------------------------------------------------------------------------

## The rest of the surface

- **State backends**: Pulumi Cloud is the default, but `pulumi login s3://bucket` / `gs://` / `azblob://` / `file://` all work. Self-hosting costs you drift detection, RBAC, deployments, and the console.
- **ESC** (Environments, Secrets, Config): hierarchical config with dynamic secrets brokering — pull short-lived AWS creds instead of storing keys. Now supports rotated secrets and versioning. Genuinely good; also usable outside Pulumi as an `esc run` env-var provider.
- **CrossGuard**: policy-as-code in Python/TS rather than Rego.
- **Migration**: `pulumi convert --from terraform` converts HCL; `pulumi import` adopts existing resources and generates code. Both get you ~80% there. Pulumi Cloud can also serve as a state backend for Terraform and OpenTofu.
- **Kubernetes**: strongest area — typed manifests, Helm chart consumption, an operator for in-cluster GitOps, `await` logic that actually waits for rollouts to be healthy.
- **AI**: Pulumi ships an MCP server, and Neo (agent) has replaced the older Copilot. Latest CLI is 3.255.x.
- **Service Provider v1.0** (May 2026): manage Pulumi Cloud itself as code — fine-grained RBAC, IDP, and audit-log export as code.

------------------------------------------------------------------------

## Where it bites

- **Ecosystem depth.** Terraform has ~10× the blog posts, StackOverflow answers, and prior art. LLMs are noticeably better at HCL than at Pulumi Python. You will read provider source more often.
- **Real languages, real messes.** Nothing stops someone from writing infra with three layers of indirection and a metaclass. HCL’s stupidity is a feature under team pressure.
- **Refactoring.** Renaming a resource’s logical name replaces it. `aliases` and `opts=ResourceOptions(...)` fix it, but it’s easy to nuke a database by renaming a variable.
- **Python typing is mediocre.** Provider SDKs are generated; type hints exist but `Input`/`Output` unions make mypy noisy.
- **Debugging.** Async graph + generated code = ugly stack traces.
- **Pricing model.** A credit is one resource managed for one hour; Team is \$0.0005/credit with 150,000 free credits monthly, covering roughly 200 resources running continuously. Kubernetes blows through this fast — one Helm chart can be 200 resources on its own. Individual is free with unlimited resources but caps deployment minutes and secrets; Enterprise is a sales conversation. Self-hosted state sidesteps the whole thing.

------------------------------------------------------------------------

## My recommendation for your situation

Given a Django/Postgres/Redis SaaS with Ansible already in play: **don’t adopt Pulumi unless you hit one of two triggers.**

1.  **Per-tenant or dynamic provisioning.** If Recovery Metrics ever needs to spin up isolated infra per clinic, Automation API from Django is the best tool that exists for it, full stop. That alone justifies it.
2.  **You’re moving to Kubernetes.** Pulumi’s k8s story beats HCL meaningfully.

If neither applies, your infra is probably a handful of managed services and VMs, and OpenTofu (or even just Ansible + a small TF root module) is the lower-risk default — better hiring pool, better AI assistance, fewer unknowns. Pulumi’s “you already know Python” pitch is real but oversold: the hard parts of IaC are provider semantics and state management, not syntax, and those are identical either way.

If you do try it: start with a self-managed S3 backend, one non-critical stack, Python, and no components until you’ve felt the Output model for a week.

------------------------------------------------------------------------

Back to top
