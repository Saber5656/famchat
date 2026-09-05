# Research: Existing kids/family messengers — comparative notes

Status: desk research (agent knowledge, 2026-07); informs safety defaults, not
time-sensitive facts. Design impact: DESIGN.md §1, §13; ADR-003, ADR-006.

## Comparison

| Product | Model | Oversight | Network shape | Lessons for famchat |
|---|---|---|---|---|
| **Messenger Kids** (Meta) | Free app, parent-managed accounts under parent's FB identity | Parent dashboard: contact approval, recent contacts, message readability (limited), remote logout | Parent-approved contacts, can extend beyond family | Validates parent-managed child accounts + device oversight; not E2EE — precedent for server-readable child messaging. Distrust of Meta's data practices is exactly the gap an OSS/self-host option fills |
| **LINE** (default family choice in JP) | General messenger | None (age-gated features only) | Open: ID search, QR, group links | The incumbent famchat replaces for children; open contact graph is the core risk |
| **Hamic / Hamic MIELS** (JP, kids' first phone) | Kids smartphone + closed messenger, parent-approved contacts | AI flagging of risky text (bullying/photos), parent notification | Closed, parent-approved | Japanese-market precedent for AI/NG flagging + parent notify; hardware-bundled, not self-hostable |
| **BAND / family group apps** | Group boards | Admin roles only | Invite-based groups | Board+chat hybrid resonates for families (famchat's 掲示板) |
| **JusTalk Kids / Kids Messenger** | Kids-only messenger | Parent controls, contact approval | Closed | Confirms category viability; closed-source, hosted-only |

## Takeaways baked into the design

1. **Parent-approved contact graph is the industry-consensus core control**
   → famchat goes further: the graph is the family space itself; no external
   contacts exist at all (DESIGN §1.4-1, §5).
2. **None of the mainstream kids messengers use E2EE**; oversight and safety
   filtering consistently win that trade-off → supports ADR-003.
3. **Notification-on-flag (Hamic style) beats hard blocking** for family
   trust dynamics → famchat default `moderation_mode=flag` with guardian
   notification; `block` remains a per-space option (DESIGN §13.4).
4. **Transparency to the child** (Messenger Kids shows kids what parents can
   see) is the ethical differentiator vs stealth-monitoring apps → famchat
   makes the oversight notice a hard acceptance criterion (DESIGN §13.3).
5. **Gap famchat uniquely fills**: no mainstream option is open-source,
   self-hostable, and family-server-owned. That is the project's reason to
   exist and shapes ADR-005 (self-host = SaaS).
