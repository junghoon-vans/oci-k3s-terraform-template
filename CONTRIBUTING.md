# Contributing

This is a template repo: nothing here is wired up to a live GitOps pipeline or
a real running cluster. CI keeps that distinction: pull requests receive
credential-free Terraform checks, while trusted planning and applying are
available only from `main` after you configure the GitHub environments below.

## Workflow

1. Branch off `main`. Use a short, descriptive branch name.
2. Change `.tf` / `.tftpl` files as needed.
3. Run locally before pushing:

   ```bash
   terraform fmt -recursive
   terraform init -backend=false -input=false -lockfile=readonly
   terraform validate
   ```

4. Open a PR. CI runs `fmt -check`, initializes providers/modules with
   `-backend=false` and the committed lockfile, then validates. It has no OCI
   credentials and does not read remote state or create a plan.
5. Merge after the checks pass. A push to `main` runs a trusted plan only if
   credentials are configured; review the workflow summary before deciding
   whether to apply.
6. To apply an explicitly approved change, manually dispatch the Terraform
   workflow on `main` with `operation=apply`. It creates a fresh plan, retains
   that exact plan artifact for one day, and waits at `terraform-apply` for
   required reviewer approval before applying it. Confirm the run succeeds and
   that the changed node(s) reach `Ready` (see
   [Verification](./README.md#verification)).

Configure `terraform-plan` to allow deployments from `main` and add its OCI
and backend secrets. Configure `terraform-apply` to allow deployments from
`main`, add the same secrets, and require reviewer approval. GitHub environment
protection is external to the workflow YAML; without required reviewers on
`terraform-apply`, the manual apply is not approval-gated. A fresh template
without `OCI_TENANCY_OCID` in `terraform-plan` skips trusted plan/apply jobs.

## What a healthy plan looks like

For a small change (e.g. resizing a boot volume, tweaking a security rule),
expect a small, targeted diff -- a handful of resources added/changed, ideally
zero destroyed. Treat any of these as a stop-and-investigate signal before
manually dispatching an apply:

- A plan that wants to **recreate resources that already exist** (e.g. the
  whole VCN or existing instances). This usually means Terraform cannot see
  the real remote state. Check that trusted Terraform init uses the intended
  OCI backend; do not approve/apply until resolved.
- Any unexpected `destroy` of `module.compute.oci_core_instance.this[...]`,
  the VCN, or the NLB.
- A plan far larger than your actual diff would suggest.


## Always Free budget

The default sizing in this template sits exactly at the Always Free ceiling
for a single tenancy -- there's no slack, so scaling up needs an explicit
tradeoff:

- **Block storage**: 4 nodes x 50 GB boot volume = 200 GB, all of the 200 GB
  Always Free allowance. Any PVC backed by an OCI Block Volume storage class
  (default minimum 50 GB) is billed. Prefer node-local storage (e.g. the
  `local-path` CSI provisioner) for anything that doesn't need to survive a
  node replacement.
- **ARM compute**: 2 x `A1.Flex` (1 OCPU / 6 GB each) = the full 2 OCPU / 12 GB
  Always Free ARM pool. A third ARM instance, or resizing an existing one
  larger, is billed.
- **AMD compute**: 2 x `E2.1.Micro` = the full Always Free AMD micro instance
  limit (2 instances). A third is billed.

PR validation has no access to OCI credentials. Trusted plan/apply uses OCI
and backend secrets scoped to both the `terraform-plan` and `terraform-apply`
GitHub environments. Keep `terraform.tfvars`, `backend.hcl`, and keys out of
git; see [README.md](./README.md#cicd) for the environment setup.

## Commit style

Conventional Commits (`feat(scope): ...`, `fix(scope): ...`, `chore(scope): ...`,
`ci(scope): ...`).
