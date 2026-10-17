---
layout: article
title: "Azure Quota Groups"
date: 2026-10-17
---

# Azure Quota Groups: Don't Let Approved VM Quota Die With a Cancelled Subscription

I like Azure Quota Groups for a simple reason: approved VM-family vCPU quota is a scarce, hard-won asset, and cancelling a subscription without a plan for that quota is how you quietly throw it away. In a world where GPU and CPU capacity in popular regions is tight, quota you already hold is worth more than quota you'd have to request again from scratch. This article is about turning "preserve quota before cancellation" into a deliberate operational step, not an afterthought.

## What Azure Quota Groups actually are

A Quota Group is an Azure Resource Manager object (`Microsoft.Quota/groupQuotas`) created at Management Group scope. It exists specifically to manage IaaS compute quota — VM-family vCPUs — at a group level instead of purely per subscription, and a single group can hold limits across multiple region and VM-family combinations. Microsoft describes this as elevating the quota construct from subscription level to a group-level object that customers can self-manage without filing a support ticket for every reallocation.

That's the headline benefit: once quota sits in a group, moving it between member subscriptions is a self-service operation, not a new capacity request.

## It does not pool quota automatically — and that matters

Here's the part I think gets misunderstood quickly: adding a subscription to a Quota Group does not automatically change or merge its existing quota. The subscription keeps carrying its existing quota and usage; subscription quota stays separate from the group limit. A brand-new Quota Group actually starts with a group limit of zero. It only gains quota through an approved group-level increase request, or through an explicit deallocation from a member subscription.

Quota moves in and out of the group through two distinct, explicit operations:

- **Deallocation** — unused quota moves from a subscription into the group.
- **Allocation** — quota moves from the group into a target subscription.

Each operation targets exactly one region and one VM family at a time. There's no documented bulk "move everything" call, so a broad quota migration across several families or regions means repeating this sequence several times.

It's also worth knowing that actual deployment-time quota checks still run against the destination subscription's own quota, not the group's pooled number. So the real goal of a transfer isn't to make quota "visible in the group" — it's to raise the receiving subscription's own limit.

## The practical sequence before you cancel a subscription

If a subscription slated for cancellation holds approved, unused VM-family vCPU quota you still need elsewhere, the documented mechanics support a straightforward checklist:

1. Register the `Microsoft.Quota` and `Microsoft.Compute` resource providers on both subscriptions.
2. Create or reuse a Quota Group, and add both the outgoing and the continuing subscription as members. Subscriptions do not need to share a Management Group — a Quota Group inherits permissions from its owning Management Group but doesn't auto-sync membership from the hierarchy, so subscriptions from elsewhere can join.
3. Deallocate the unused quota from the outgoing subscription into the group, scoped to the specific region and VM family.
4. Allocate that quota from the group to the surviving subscription.
5. Validate the resulting quota on the surviving subscription and check the group's snapshot.
6. Remove the outgoing subscription from the group once it's no longer needed.
7. Only then proceed with cancellation.

This is a synthesized operational sequence built from the documented atomic operations — not a Microsoft-published runbook — but each step traces back to a real, documented capability.

## Why this deserves to be a governance gate, not a courtesy step

Here's the uncomfortable fact driving this whole article: Microsoft's current documentation does not describe what happens to quota, or to a Quota Group's allocation record, if a subscription is cancelled before quota is deallocated. It also doesn't describe any recovery process for quota after a subscription is already gone. That silence is not a guarantee either way — it's simply unaddressed in the pages I checked. Given that, I'd treat "verified transfer before cancellation" as a required change-management gate, not an optional nicety. If the documentation doesn't promise recovery, don't bet your approved quota on an undocumented safety net.

This fits cleanly under the Operational Excellence pillar of the Well-Architected Framework: a deliberate hold-and-verify step before an irreversible action (subscription cancellation) is exactly the kind of guardrail that separates mature platform operations from "we'll figure it out later."

There's a cost angle too. Re-requesting capacity-constrained VM-family quota from scratch can mean waiting on Microsoft approval again, in a market where that specific SKU or region might be constrained. Redistributing quota you already hold sidesteps that queue entirely — provided you act before the subscription disappears.

## Governance and separation of duties

Because Quota Group membership is independent of Management Group placement, a subscription from a completely different Management Group can join a group it doesn't "belong to" organizationally. That's convenient, but it means platform teams need a clear naming and ownership convention for who owns which Quota Group and who may add subscriptions to it — separate from the existing Management Group hierarchy used for policy and RBAC.

The permission model backs this up with a deliberate split: creating or deleting a Quota Group requires the **GroupQuota Request Operator** role at Management Group scope, while day-to-day quota operations on member subscriptions need **Quota Request Operator** plus **Reader**. That's a sensible separation of duties — the people who can spin up or tear down groups aren't necessarily the same people moving quota around inside one — and it belongs in your existing privileged-access reviews, since these roles can move capacity entitlements across cost centers or business units.

## What's the catch?

A few things keep me from calling this a solved problem:

- **Scope is narrow.** This only covers IaaS compute quota, only in public cloud regions, and only for Enterprise Agreement, Microsoft Customer Agreement, or Internal subscription types. CSP and MSDN-type subscriptions aren't eligible by this mechanism.
- **Quota is not capacity.** Even a successful allocation only raises a subscription's approved limit. A Quota Group increase request can still be rejected if Azure capacity is genuinely constrained in that region and VM family, and a deployment can still fail at runtime even with quota approved, because the actual regional/zonal capacity isn't guaranteed by quota alone.
- **Region and zonal access is separate.** Quota Groups manage limits, not access. If a subscription doesn't already have the necessary region or zonal access, a transfer or a deployment relying on that quota will still fail.
- **Cross-tenant membership is undocumented.** The pages I reviewed confirm that subscriptions from a different Management Group can join a group, but say nothing about subscriptions in a different Microsoft Entra tenant. Don't assume it works, and don't assume it doesn't — validate it directly if that's your scenario.
- **No explicit maturity label.** The documentation I reviewed carries no preview or GA banner at all; it reads as standard how-to content with dated metadata, but that absence isn't itself a GA statement.
- **Cleanup discipline matters.** Deleting a Quota Group requires removing all member subscriptions first, and deleting the Management Group that owns a Quota Group causes loss of access to its limit — the documented mitigation is allocating quota out and deleting subscriptions/the group before Management Group deletion, or recreating the Management Group with the same ID afterward.

## Would I try it? Where, and how

Yes, but as a controlled, low-stakes experiment first, not a production rollout. I'd pick a test Management Group and two non-production subscriptions that already hold a small amount of quota in the same VM family and region. I'd run a single deallocate-then-allocate cycle, then check three things: the destination subscription's own quota value actually changed, the group's snapshot reflects it correctly, and a VM of that family can genuinely deploy up to the new limit in that region. Only after that would I build a cancellation checklist around it for real decommissioning work.

Cost and cleanup caution: Quota Group objects and membership don't appear to carry their own Azure charges, but leaving subscriptions in a group, or deleting a Management Group while the group still holds a nonzero limit, creates administrative lock-in. Follow the documented removal order — remove subscriptions, delete the group, then deal with the Management Group — and you'll avoid a cleanup headache.

The bigger validation step, though, is organizational: before you lean on this as your pre-cancellation safety net, confirm directly with current Microsoft documentation or support whether newer guidance addresses quota handling after subscription cancellation. The pages I checked are silent on it, and silence is not a plan.

## References

- [Share Quota Across Subscriptions with Azure Quota Groups - Azure Quotas | Microsoft Learn](https://learn.microsoft.com/en-us/azure/quotas/quota-groups)
- [Transfer quota within an Azure Quota Group - Azure Quotas | Microsoft Learn](https://learn.microsoft.com/en-us/azure/quotas/transfer-quota-groups)
- [Create and delete Azure Quota Groups to manage quota for a group of subscriptions - Azure Quotas | Microsoft Learn](https://learn.microsoft.com/en-us/azure/quotas/create-quota-groups)
- [Add or remove subscriptions from an Azure Quota Group - Azure Quotas | Microsoft Learn](https://learn.microsoft.com/en-us/azure/quotas/add-remove-subscriptions-quota-group)
