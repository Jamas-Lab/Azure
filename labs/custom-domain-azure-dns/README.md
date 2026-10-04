# Custom domain with Azure DNS delegation

**Exam area:** AZ-104, Manage Azure identities and governance; Implement and manage virtual networking (Azure DNS)
**Run on:** 4 Oct 2026
**Status:** Complete

## Goal

Take a real domain, `jamalabs.co.uk` (registered at Fasthosts), host its DNS in Azure DNS, and use that zone to prove ownership of the domain to Microsoft Entra ID. Along the way, show that creating a zone and delegating a domain are two different things.

## Design

| Item | Choice |
|---|---|
| Domain | `jamalabs.co.uk`, registrar Fasthosts |
| DNS hosting | Azure DNS public zone in a long-lived networking resource group |
| Verification method | TXT record at the zone apex |
| Tooling | Azure CLI, Microsoft Graph through `az rest`, and `dig` for every DNS check |
| Cost | £0.38 a month for one hosted zone (Azure pricing calculator, GBP). The per-zone charge is calculated daily; query charges are negligible at lab volume |

The break-glass admin account stays on the tenant's built-in domain, and the new domain is not set as the default.

## What was run and what happened

| Stage | Change | Expected | Result |
|---|---|---|---|
| 0 | Baseline with `dig`, no changes | Find out whether anything is live | The registrar's three nameservers. The apex and `www` resolved to a parking page. No AAAA, MX or TXT records, and no DS record (no DNSSEC). The registrar's DNS panel listed zero records, so its default parking records were not shown there |
| 1 | Created the Azure DNS zone | Zone exists, but the internet still asks the registrar | Zone created with four Azure nameservers, one each under `.com`, `.net`, `.org` and `.info`. Public DNS still returned the registrar's nameservers, while Azure's nameserver answered an SOA query when asked directly |
| 2 | Added the domain to Entra ID through Graph | Domain appears unverified, with a verification record to place | Domain added, unverified, authentication type Managed. Graph offered a TXT record and an MX record for verification. The MX had the lowest priority and pointed at a `.invalid` host, so it could never route mail |
| 3 | Put the TXT value in the Azure zone, then tried to verify | Verification fails, because the domain is not delegated yet | TXT record created. A direct query to Azure's nameservers returned it (empty on the first try, seconds after creation; present on a recheck). The verify call failed with `Bad Request`, detail `TargetHostCannotBeResolved` |
| 4 | Changed the domain's nameservers at the registrar to the four Azure ones | Hours to take effect (a third-party guide quoted 3 to 6 hours, up to 72) | `dig +trace` showed the `.uk` registry servers returning the four Azure nameservers within about two minutes. A plain `dig` through the local resolver still returned the old nameservers from cache |
| 5 | Ran the verify call again, nothing else changed | Verification passes | `isVerified` true. The Entra portal agreed. The tenant's default domain was unchanged |
| 6 | Created copies of four lab users with sign-in names on the new domain, using a script that is a dry run unless told otherwise | Accounts created; as new objects they inherit no roles or group memberships | Created, then display names corrected. One account was seen in the portal; a CLI listing of all four was not recorded |

## What this lab showed

- **A zone is not live until the registrar delegates to it.** Before delegation, Azure's nameservers answered when asked directly, but the rest of the internet still asked the registrar's.
- **Entra ID proves ownership with a TXT or MX record.** An A record is never a verification type.
- **The registrar's panel is not the full truth.** It listed no records while public DNS served a parking page address. `dig` shows what the internet sees.
- **"Propagation" is two timers.** The registry update took about two minutes here; resolver caches can hold the old answer for the delegation's TTL, which is 172,800 seconds (48 hours).
- **Verifying a domain does not make it the default** for new users.
- **Copied users are new objects.** Role assignments and group memberships belong to the original objects and do not follow a copy.

## Commands

Placeholders in angle brackets. Run in Bash.

```bash
# Stage 0: baseline
for t in NS A AAAA MX TXT; do echo "== $t"; dig $t <domain> +short; done
dig DS <domain> +short

# Stage 1: zone and its nameservers
az network dns zone create --resource-group <resource-group> --name <domain>
az network dns zone show --resource-group <resource-group> --name <domain> --query nameServers --output tsv

# Stage 2: add the domain to Entra ID and read the verification records
az rest --method post --url "https://graph.microsoft.com/v1.0/domains" --headers "Content-Type=application/json" --body '{"id":"<domain>"}'
az rest --method get --url "https://graph.microsoft.com/v1.0/domains/<domain>/verificationDnsRecords"

# Stage 3: TXT record in the zone, value read straight from Graph
TXT=$(az rest --method get --url "https://graph.microsoft.com/v1.0/domains/<domain>/verificationDnsRecords" --query "value[?recordType=='Txt'].text | [0]" --output tsv)
az network dns record-set txt add-record --resource-group <resource-group> --zone-name <domain> --record-set-name "@" --value "$TXT"

# Stage 4: after changing the nameservers at the registrar
dig +trace +nodnssec NS <domain> | tail -n 15

# Stage 5: verify
az rest --method post --url "https://graph.microsoft.com/v1.0/domains/<domain>/verify"
```

## Cleanup and what was kept

- **Kept:** the Azure DNS zone (the domain is now hosted in Azure DNS) and the four lab users on the new domain.
- **Undo path, if ever needed:** set the registrar back to its default nameservers.
- **Left alone:** an older, unverified domain entry in the tenant that is not part of this lab.

## Left out on purpose

Tenant and subscription IDs, the verification token, and user sign-in names.
