## Purpose

This document evaluates three candidate products — Azure DevOps, Harness FME, and Smartsheet — for governing the creation of feature flags. It is prepared as input for architectural review: a proposal to inform a decision, not a design commitment.

The objective is to establish governance over flag creation while assessing what each product does and does not provide.

## Governance parameters

We define a typical governance capability as one that answers the following parameters. 
The choice of tooling depends on what we aim to achieve against each parameter and which ones are our priority.

**Should the flag exist?**
A sanction decision precedes the flag. 
Dev teams raise the need for a feature flag. 
Architects make the approval decision; the flag is approved or rejected.

**Approval workflow**
Feature flag requests should move through a defined approval process.
- *Authority* — who is entitled to approve.
- *Separation of duties* — four eyes; the requester cannot be the approver (requester ≠ approver).

**Enforcement**
Enforcement at the point of creation — a flag is created only if it was approved through the governance process. 
Exceptions are permitted for genuine reasons, through a defined override path.

**Auditability**
An audit record that captures who, when, under what role, and the before/after state.

**Drift detection**
The sanctioned state (the registry) and the runtime state (the flag service) live in different systems, so drift between them is possible and must be detectable.

## How to read this document

Each product is assessed against the five parameters above, in the same order, so the three can be compared parameter by parameter. Where a capability is native, that is noted; where it must be built, that is called out

The analysis follows.


# Azure Devops

Should the flag exist?

    A work item - 

        Create an AzDo work item, typically serves as an intake frorm which teams can use to submit feture flag requests.
        Azdo Serves as the system of record.

    Work item states - 

        Used as the workflow.
        Draft > Request for Approval > Approved | Rejected

Approval workflow
   
    RBAC - 

        Two disjoint groups - Requesters & Approvers
        4 eyes; Requesters can't be approvers themselves.
        Workflow is triggered via changing Work item states. 

Enforcement at the point of creation. 

    Shift governance to pre-deployment. 
    Block pull requests if Feature Flags did not pass governance

    Blocking pull requests based on work item status is not Native to Azdo.
    But, shall be achieved through - Pull reqeust workflow extensibility

        Feature flag work items are linked to the PR.
        An alternative to this is for the PR workflow extension to read the flag id from JSON and query AzDo for the work item.
        During pull request, AzDo is polled for the work items' status.
        When the feature flag work items are not approved, PR is blocked.
        
        This is how Governance can be enforced.

    Allow Pull Requests - on an exception basis.

        When Feature flags did not pass governance, and teams need the deployment to go through, 
        A manual override shall allow the PR to complete rather than blocked.

Auditability — 

    The work item in AzDo is the system of record.
    It carries the audit information, such as, Created By, Approved By etc. 
    Any discussions happened while approving or denying the feature flag work item are available in the work item's  conversations area.

Reconciliation - (optional)

    Drift is possible since AzDo and the Feature Flags Service's runtime store are different systems.
    Drift detectioin is not natively offered by AzDo. It needs to be built if we want it. Hence marked as optional.

Constriants:

    AzDo is project based.
    Intela and SDT are separate projects in AzDo, this means the flag registry and work items should be managed separately under Intela and SDT projects.

---

# Harness FME

Hanress is a fully featured product that can manage Feature flags. 
SaaS by default (single control plane).  
Self-Managed Enterprise Edition available if data residency requires.

Governance is a component of it's features, but is applied for changes to the flag, not while creating a flag.
A person that is designated and/or authorized to create feature flags is allowed to create the flag in the system.
There is no prior step to this such as making a request for a feature flag and someone should approve it.

When a feature flag is created, it can contain certain information we are looking to capture through the governance process. 

I see this as a potential replacement for our Feature flag service, if we decide to avail it's feature flags management capabilities, but with potential design changes to either the feature flags service or the downstream applications that consume the feature falgs service. But note, this needs further analysis.

I don't think FME can be used as a layer above our feature flags service, like AzDo can. 
It will be replacing the feature falgs service, which shall be considered as the value of the investment.


Should the flag exist?

    Intake / system of record:

        No native "flag request" that exists before the flag creation.
        Authority is governed by RBAC, which defines User groups and Users.
        A person in an authorized role may create the flag.
        Once created, The feature flag entity itself (project-scoped) is the system of record.

    Workflow states — 

        Not applicable. There is no intermediate state that exists before flag is created.

    Approvals - 

        There is no intermediate approval that needs to happen before a flag is created.
        Authority is governed by RBAC — a person in an authorized role may create the flag.
        Approvals kick in when changes are to be made to the feature flag state, not at the time of flag creation.

    In essence, Harness simplifies this flag creation process.


Enforcement at the point of creation. 

    FME governance is applicable for changes to the flag state, not at the time of creation.
    There is not a concept of blocking pull requests before or during Flag creation.
    
    A native AzDO integration exists, but it provides work-item < > flag linking so you can see which work pertains to which flag — not approval-gating on work-item state. 


Auditability 

    Per-environment audit logs, change-request records, policy-evaluation history, archiving preserves history, and a Webhook (Audit Logs) export.


Reconciliation

    If FME is adopted as the flag runtime (native SDK model): registry and runtime are the same system — no registry-vs-runtime drift, so reconciliation is not needed.


Constraints:

    Approvals come into play only when someone wants to change the state of the flag. Not during the flag is created.

    Consuming feature flags from Harness FME requires using their SDK.
    The scope of change for this adoption - needs to be assessed.
    One option is to examine if the feature flag service can be made as a wrapper around FME such that downstream applications doesn't need to change. 

    Scope of migration needs to be assessed as well.


Additional Insights for Harness

Harness FME is architected to support teams and organizations of any size, from a single developer to multiple value-stream enterprises. 

Harness uses the below mentioned structural constructs to organize feature flags.

Accounts
Projects
Environments
Traffic Types
Tags
Segments
RBAC (User Groups & Users at the control plane)
Targeting Rules
Statuses

A typical future state using Harness for SDT and Intela might look like this as follows:

Account (Tenant; eg. Deloitte Tax) (single control plane)
│
├── Project: INTELA
│   ├── Traffic types ...... user, account
│   ├── Feature flags ...... (e.g. data-prep-tab), (project-scoped)
│   ├── Segments
│   └── Environments        approvals   gate
│       ├── Dev-Intela          off      soft
│       ├── QA-Intela           off      soft
│       ├── INT-Intela          on       hard
│       ├── Staging-Intela      on       hard
│       └── Prod-Intela         on       hard
│
└── Project: SDT
    ├── Traffic types ...... user, account
    ├── Feature flags ...... (e.g. checkout-document), (project-scoped)
    ├── Segments
    └── Environments        approvals   gate
        ├── Dev-SDT             off      soft
        ├── QA-SDT              off      soft
        ├── INT-SDT             on       hard
        ├── Staging-SDT         on       hard
        └── Prod-SDT            on       hard


Policy as code 

    Harness has a concept called policy as code.
    A policy is a rule, written as code in the OPA Rego policy language, which is designed based on Open Policy Agent OPA https://www.openpolicyagent.org

    Harness uses the Harness OPA server managed by Harness to evaluate these rules.

    Policy as code can be stored directly in the OPA service in Harness 
    or use the Git Experience to store policies in a Git repository.

    These rules are evaluated whenever a feature flag or definition is created, updated, deleted, or archived.

    Note - Policy as code is a paid feature, not available in free tier.

---

Smartsheet

Should the flag exist?

    Intake Form - 

        Used to request feature flags
        A sheet Serves as the system of record.

Approval workflow - 
        
    Built via Automation → "Request an approval when specified criteria are met." 
    It pauses until the approver approves or declines, then writes a value into a status column. 
    Approver is set either as a named contact or pulled from a contact column on the row; 

    Status is just a column value. 
    It changes in the following ways: 
        The approval automation writes it on approve/decline (configurable submitted/approved/declined values); 
        A person with edit access types it directly in the cell; 
   
    RBAC - 

        Smartsheet has sharing permissions (Owner/Admin/Editor/Commenter/Viewer, item-scoped) and shareable groups, 
        but there is no native rule enforcing requester ≠ approver.

Enforcement at the point of creation - 

    Smartsheet cannot gate flag creation. Enforcement must still be built externally.
    A pull request extension queries the Smartsheet REST API, reads the flag's row by flagId, checks the status column, and blocks the PR if the flag is not approved.


Auditability — 

    Three layers: 
    Cell history (per-cell who + when, viewable by anyone with Viewer+); 
    Activity log (all changes, deletions with data, who viewed, sharing changes — filterable/exportable, but Business/Enterprise paid only); 
    Event Reporting


Reconciliation - (optional)

    Drift is possible since Smartsheet and the Feature Flags Service's runtime store are different systems.
    Drift detectioin is not natively offered by Smartsheet. It needs to be built if we want it.




