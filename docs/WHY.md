# Why This Exists

## The Mapping Table Is A Trap

Every multi-cloud discussion opens with the same table. VPC equals VNet. Security group equals NSG. Subnet equals subnet.

It's true at a glance and dangerous in practice — and I wanted to know exactly where it breaks, because the answer determines whether a cross-cloud migration is a find-and-replace or a redesign.

The only way to find out is to write the same architecture twice and deploy both.

## What I Found

The mental model transfers almost entirely. CIDR planning, blast radius, network segmentation — provider-agnostic. If you can reason about one, you can reason about the other.

Three things genuinely don't transfer:

**1. Resource Groups are mandatory in Azure and absent in AWS.**
Not an organisational preference — a lifecycle boundary. It changes how deletes cascade, how RBAC scopes, and how cost rolls up.

**2. NSG rules are priority-ordered and support explicit denies. Security groups do neither.**
AWS security groups are pure allow-lists with no precedence. Azure evaluates by priority, first match wins, denies exist. **A rule-for-rule port of a non-trivial security group will silently behave differently.** This is the migration trap, and it doesn't surface in a mapping table — it surfaces in production.

**3. Attachment model differs.** AWS binds a security group to a resource's ENI. Azure binds an NSG to a subnet or NIC. Same intent, different blast radius — a subnet-level NSG governs everything inside it by default.

## Why That Third Point Matters To Me

I spent two and a half years operating a cloud-native 5G core in production — on-call at 99.9% SLA, and executing wave-based 4G-to-5G cutovers with a rollback path at every wave.

The thing that breaks a migration is never the resource that fails loudly. It's the one that comes up *healthy* and behaves differently than the thing it replaced. A ported security rule that quietly evaluates in a different order is exactly that failure mode — validated green, wrong in production.

That's why I went looking for the divergences rather than the equivalences. The equivalences are in the docs. The divergences are what cost you an outage.

## Deliberate Scope

Network foundation only. No compute, no remote backend, no pipeline gating.

That's a choice, not an omission. Adding a VM would have introduced cost and noise without changing what the project answers. Every resource here is free-tier; teardown runs after every apply. The next iteration — remote state with locking, shared modules, plan-on-PR with gated apply — is tracked in the README.

## What Comes Next

Same discipline, more surface: remote state backends on both clouds, then a wave-based migration scenario with validation and rollback — the pattern I ran at carrier scale, expressed in IaC.
