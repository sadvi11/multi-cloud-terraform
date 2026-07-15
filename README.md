# Multi-Cloud Infrastructure with Terraform

**One codebase. Two clouds. Identical network architecture.**

Deploys the same network foundation to AWS and Azure — not to prove Terraform works, but to document exactly where the two clouds stop being interchangeable.

## The Architecture

| Layer | AWS | Azure |
|---|---|---|
| Container | *no equivalent* | **Resource Group** |
| Network | **VPC** — 10.0.0.0/16 | **Virtual Network** — 10.0.0.0/16 |
| Subnet | **Subnet** — 10.0.1.0/24 | **Subnet** — 10.0.1.0/24 |
| Firewall | **Security Group** — HTTP :80 | **NSG** — HTTP :80 |
| Region | ca-central-1 | Canada Central |

Both sides deployed and verified in their respective consoles.

## Why This Exists

My production experience is AWS. Most "multi-cloud" claims mean "I read the Azure docs." I wanted the honest version: write the same architecture twice, deploy both, and find out where the abstraction leaks.

It leaks in three specific places.

## Where the Clouds Actually Diverge

The mapping table makes it look like find-and-replace. It isn't.

**1. Azure mandates Resource Groups.**
Every Azure resource must live inside one — a required container with its own lifecycle. AWS has no equivalent; you organise with tags and accounts. Not cosmetic: it changes how you scope deletes, permissions, and cost tracking.

**2. NSG rules are priority-ordered. Security Groups aren't.**
Azure evaluates NSG rules by priority number — lower wins, first match applies, and deny rules exist. AWS Security Groups are pure allow-lists: no ordering, no precedence, no denies. A rule-for-rule port of a non-trivial SG will not behave the same way.

**3. The attachment model differs.**
AWS attaches a Security Group to a resource (an instance's ENI). Azure attaches an NSG to a subnet or a NIC. Same intent, different blast radius — a subnet-level NSG governs everything inside it by default.

These are the things you only learn by deploying both.

## Structure

    multi-cloud-terraform/
    ├── aws/          # aws provider — VPC, subnet, security group
    ├── azure/        # azurerm provider — RG, VNet, subnet, NSG
    └── docs/

Each cloud has an isolated provider config and state, so they deploy independently:

    cd aws   && terraform init && terraform apply
    cd azure && terraform init && terraform apply

## Engineering Notes

**State is gitignored.** State files contain resource IDs and subscription identifiers — they never belong in version control. Production would use remote state: S3 + DynamoDB locking on AWS, or an Azure Storage Account with blob leasing.

**Free-tier by design.** No compute — networks, subnets, and firewall rules cost nothing on either cloud. Teardown runs after every test regardless; the habit is what protects you when the next project does have a VM in it.

**Scope, honestly:** this is a network foundation, not a production landing zone. No compute, no remote backend, no CI/CD gating. Those are the next iteration — the point of this one is the cross-cloud comparison.

## Next

- [ ] Remote state backends (S3 / Azure Storage) with locking
- [ ] Shared modules to reduce per-cloud duplication
- [ ] CI/CD: plan on PR, gated apply on merge

**Stack:** Terraform · HCL · AWS · Azure
