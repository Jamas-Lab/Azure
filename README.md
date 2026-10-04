# Azure labs

Hands-on Azure labs - each lab is written up the same way: what I built, what I expected, what actually happened, and what I did not run.

## Reading the status column

- **Complete**: every stage was run and recorded.
- **Partial**: some stages were run. The write-up lists which ones were not.
- **Planned**: designed, not yet run.

## Labs

| Lab | Exam area | What it shows | Status | Date |
|---|---|---|---|---|
| [Hub-and-spoke with an NVA and a cross-region spoke](labs/hub-spoke-nva/) | Virtual networking | Non-transitive peering, effective routes, a user-defined route to an NVA, global peering latency | Partial: stages 0 to 5 run | 1 Oct 2026 |
| [Custom domain with Azure DNS delegation](labs/custom-domain-azure-dns/) | Identity and governance; Azure DNS | Delegating a real domain to Azure DNS, verifying it in Entra ID through Microsoft Graph, users on the custom domain | Complete | 4 Oct 2026 |

## Conventions

- Region is UK South unless a lab says otherwise.
- Results are what I recorded. Expected behaviour that I did not test is labelled as reasoning.
- Identifiers such as subscription, tenant and user IDs are left out.
