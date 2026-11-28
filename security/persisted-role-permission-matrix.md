# Persisted tenant role-to-permission matrix

**Status:** Source-derived implementation snapshot, reviewed 2026-11-28.  
**Source of truth for this snapshot:** `HeapCare.Api/src/HeapCare.Application/Tenancy/TenantPermissionCatalog.cs`, `HeapCare.Api/src/HeapCare.Api/Security/TenantRolePermissions.cs` (compatibility adapter), and `HeapCare.Api/src/HeapCare.Api/Middleware/TenantContextMiddleware.cs`.  
**Important:** This records what the current API implementation grants. It is not, by itself, product-owner approval of the intended job responsibilities, a clinical privilege model, or proof that every endpoint and resource scope is correctly enforced.

## Trust and evaluation model

For a tenant-scoped request, the API validates the OIDC token, then resolves the validated issuer and subject against an active persisted membership in the selected active tenant. The persisted role is passed to the role-to-permission resolver. The API removes caller-supplied reserved role/permission claims and creates a dedicated `HeapCareTenantMembership` identity containing the derived permission claims. Authorization policies accept permissions only from that dedicated identity.

The `X-Tenant-ID` header selects tenant context; it does not grant access. Hospital and department scopes are separately enforced through organization-scope context and service/persistence checks.

## Current persisted tenant roles

The table below is a readable summary of the permission constants returned by the current resolver. “—” means the resolver currently returns no permission in that capability group.

| Capability | HospitalAdmin | FacilityManager | FacilityViewer | PatientRegistrar | ClinicalUser |
|---|---|---|---|---|---|
| Facility hierarchy | Read, create/update, delete | Read, create/update | Read | Read | Read |
| Membership administration | Manage | — | — | — | — |
| Audit events | Read | — | — | — | — |
| Patient records | Search/read, identity read, register, duplicate review | — | — | Search/read, identity read, register, duplicate review | Identity read |
| Patient identifiers / MRN | Read, assign, policy read/manage | — | — | Read, assign | — |
| Staff profiles and assignments | Read, manage | — | — | — | — |
| Appointment availability | Read, manage | — | — | Read | — |
| Appointments | Read, manage | — | — | Read, manage | — |
| OPD queue | Read, manage | — | — | Read, manage | Read |
| OPD encounters | — | — | — | — | Read, manage, sign, amend |
| Billing catalogue | Manage | — | — | — | — |
| Billing invoices | Read, create/manage | — | — | Read, create/manage | — |
| Billing payments | Record/manage | — | — | Record/manage | — |
| Billing reports | Read | — | — | Read | — |

### Exact implementation details

- `HospitalAdmin`: `facility.read`, `facility.write`, `facility.delete`, `membership.manage`, audit read; patient read/identity-read/register/duplicate-review; patient identifier read/assign/policy-read/policy-manage; staff read/manage; availability read/manage; appointment read/manage; queue read/manage; billing catalogue-manage, invoice-read/invoice-manage, payment-manage and report-read.
- `FacilityManager`: `facility.read` and `facility.write`.
- `FacilityViewer`: `facility.read`.
- `PatientRegistrar`: facility read; patient read/identity-read/register/duplicate-review; identifier read/assign; availability read; appointment read/manage; queue read/manage; invoice read/manage; payment manage; billing report read.
- `ClinicalUser`: patient identity-read; facility read; queue read; OPD encounter read/manage/sign/amend.

The current resolver contains five tenant roles. `PlatformOperator` is a separate global operator identity and authorization policy, not a sixth tenant membership role. Its access must be reviewed independently from this table.


## IPD permissions — defined, not granted

The API defines six dedicated IPD permission identifiers, but the current `TenantRolePermissions.GetPermissions` resolver does not grant any of them to HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar or ClinicalUser:

- `inpatient.read`
- `inpatient.admit`
- `inpatient.bed.assign`
- `inpatient.bed.transfer`
- `inpatient.admission.cancel`
- `inpatient.admission.discharge`

PlatformOperator is a separate global identity and receives no implicit tenant clinical access. HeapCare.Local PR #46 passed real-Keycloak negative authorization verification for all five persisted tenant roles and the platform-only operator across the five current IPD HTTP routes. This is negative-access evidence only; it does not approve a positive role mapping.

**Approval is still required:** use the [IPD role and workflow approval worksheet](../product/decisions/ipd-role-and-workflow-approval-worksheet.md) to record hospital-approved role/operation pairs, resource scope, admission/bed sequencing, mandatory fields, cancellation and discharge rules. Until those decisions are approved and implemented through a separately reviewed change, every IPD operation remains denied to existing roles.

## Endpoint-policy distinctions and known caveats

- The role grant list is not the same thing as endpoint coverage. The [API authorization policy inventory](authorization-policy-inventory.md) records named policies and endpoint assignments.
- A permission at the HTTP boundary does not replace resource-level hospital/department scope validation in application services and persistence.
- Billing catalogue listing currently uses the `BillingInvoiceRead` policy; there is no separate catalogue-read permission in the current policy resolver. Treat this as an explicit design decision to review, not as an implied grant.
- `PatientRegistrar` currently receives duplicate-candidate review and patient identity-read permissions. Confirm that these capabilities match the approved reception workflow and minimum-necessary access requirements.
- `ClinicalUser` currently receives encounter amendment permission in addition to read/manage/sign. Clinical amendment must remain governed by the encounter's signed-state, reason, attribution and audit controls.
- Staff read/manage is currently limited to `HospitalAdmin`; do not broaden it without an approved role design and negative authorization tests.
- UI navigation and visibility are usability controls only. Every API operation must enforce its own authorization and resource scope.

## Regression coverage merged through 2026-11-27

- API PR #90 adds PostgreSQL-backed facility read/create/delete checks for all five persisted tenant roles and verifies denied mutations do not change hospital state. CI and Docker Integration Verification passed.
- API PR #92 adds negative billing coverage for FacilityViewer and ClinicalUser across catalogue, invoice, payment and report reads/writes. It verifies invoice state remains unchanged and the denied operations are attributed in audit metadata. CI and Docker Integration Verification passed.
- API PR #93 adds negative scheduling coverage for FacilityViewer and ClinicalUser across availability listing, slot retrieval and appointment booking. It verifies denied booking leaves the appointment list unchanged and denials are audited. CI and Docker Integration Verification passed.
- API PR #94 adds PostgreSQL-backed staff profile and assignment denials across FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser. It merged on 2026-11-27 as `e72af18ac6b70aef462d8d554e9f6d66c030cd63` after hosted PR CI, push CI and Docker Integration Verification passed.
- API PR #95 adds PostgreSQL-backed patient identifier/MRN authorization coverage. FacilityManager, FacilityViewer and ClinicalUser are denied identifier-policy read/manage and identifier read/lookup/assignment; PatientRegistrar retains identifier read/assignment but is denied policy read/manage. The tests verify denied operations are audited and leave policy and identifier state unchanged. It merged on 2026-11-27 as `62e73a0577d0dd10e69a1d480290d3c1e3210f30` after hosted PR CI, push CI and Docker Integration Verification passed.

- HeapCare.Local PR #29 adds a real-Keycloak/PostgreSQL acceptance harness for synthetic identities across all five persisted tenant roles and a separate PlatformOperator identity. It verifies server-derived permission claims, tenant hospital-read access and positive/negative patient-search access. It merged on 2026-11-27 as `93ae61f4742c9e5caf41ef4088bcb8a832a5dbb4`; pull-request run [#45999845006](https://github.com/HeapCare/HeapCare.Local/actions/runs/45999845006) passed both local tooling validation and full nine-service Docker/API telemetry verification. This supplements but does not replace endpoint-level PostgreSQL authorization tests or the manual authenticated accessibility review.

These are focused regression slices, not exhaustive proof of every permission, endpoint, data scope or UI route. Continue with positive/negative coverage for each capability and organization scope, including tenant, hospital and department boundaries.

## Required verification before pilot

1. Product and clinical owners approve the responsibilities and minimum-necessary access for each role, especially patient identity, duplicate review, identifiers, billing payments/reports and encounter amendment.
2. Maintain positive and negative PostgreSQL integration tests for every role/capability combination, including denied operations that must not mutate state.
3. Test tenant-wide, hospital and department scopes, including cross-tenant, cross-hospital and same-hospital cross-department access.
4. Verify that token-supplied role and permission claims cannot elevate any persisted role.
5. Verify audit attribution and metadata-only logging for allowed and denied sensitive operations.
6. Reconcile this document whenever `TenantRolePermissions`, authorization policies, endpoint assignments or role provisioning changes.

This snapshot does not lift the production/pilot HOLD. Authenticated Keycloak acceptance, full authorization regression, operational controls and independent security/privacy review remain release gates.


### Organization-scope regression coverage — patient identifiers (2026-11-27)

- API PR #96 merged as `41533bf7dfe34cc16fdd3f90aa99c1b3726ecc7f`. PostgreSQL-backed tests verify hospital-scoped HospitalAdmin access within its assigned hospital and denial of cross-hospital policy/identifier operations. Department-scoped HospitalAdmin is denied hospital-wide MRN policy, identifier listing, lookup and assignment operations.
- Denials use non-disclosing 404 responses, expose no patient/MRN data, are attributed as rejected audit events, and leave policy/identifier state unchanged. Hosted PR CI, push CI and Docker Integration Verification passed.
- This is focused coverage for the patient-identifier boundary; continue validating tenant, hospital and department scopes across each remaining domain and operation.


### OPD encounter role-denial regression — 2026-11-27

- API PR #97 merged as `ac99bd91602e03d3e6cd483a8bdba0971395eca7`. PostgreSQL-backed tests cover FacilityManager, FacilityViewer and PatientRegistrar denial for encounter read (by encounter and queue), create, note update, sign, amend, close and cancel.
- Denials return 403 without disclosing clinical note content; persisted encounter state remains unchanged; denied audit records include tenant, validated issuer/subject and HTTP status attribution.
- Hosted Release build/test, push build/test and Docker Integration Verification passed. ClinicalUser access remains separately granted and organization-scoped; the test is not a clinical acceptance or regulatory-compliance claim.


### Additional persisted-role regression coverage — 2026-11-27

- API PR #98 merged as `71fb901eafdde5e557c7b3257070317cf8c9c390`: FacilityManager is included in negative scheduling/availability and billing matrices; FacilityManager and FacilityViewer are denied patient registration and patient search. No unauthorized billing state change is permitted; the full Release, push and Docker checks passed.
- API PR #99 merged as `45dfdcb726a85df04a1daf9e76d96dfdd25a2014`: HospitalAdmin is explicitly denied OPD encounter read, creation, note update, signing, amendment, closure and cancellation. The test checks no clinical note disclosure, unchanged encounter state and denied audit attribution. Release, push and Docker checks passed.
- API PR #100 merged as `0a8f326cea2be23984a68e90e100188b311b7a4b`: hospital- and department-scoped HospitalAdmin memberships cannot query or export tenant-wide audit events despite having the persisted audit-read permission. The service-level tenant-wide scope guard returns 403. Release, push and Docker checks passed.

These additions close specific gaps in the five-role matrix. They do not replace the full authenticated Keycloak/API/PostgreSQL acceptance matrix or product/clinical approval of role responsibilities.


### Source-path and latest regression review — 2026-11-27

The source paths above were verified against the renamed HeapCare.Api main tree. Additional merged regression slices are recorded in this matrix: PR #96 covers hospital- and department-scoped HospitalAdmin restrictions for patient identifiers; PR #97 covers non-clinical-role denial for OPD encounter operations; PR #98 expands FacilityManager scheduling/billing denial coverage and patient search/registration denials for FacilityManager/FacilityViewer; PR #99 explicitly denies HospitalAdmin access to OPD encounter operations; and PR #100 denies tenant-wide audit query/export for hospital- and department-scoped HospitalAdmin memberships. These additions strengthen focused cases but do not establish exhaustive positive/negative coverage for every policy, endpoint and scope combination.


### Patient duplicate-review negative authorization — 2026-11-27

- API PR #117 merged as `0407dcfeefdd16d63299e74ba8cc27a9e3ef5e86`. PostgreSQL-backed tests verify FacilityManager and FacilityViewer receive HTTP 403 when invoking patient duplicate-candidate review. Responses do not disclose the submitted patient names; denied audit events retain tenant, validated issuer/subject, hospital resource, outcome and HTTP status attribution.
- PatientRegistrar and HospitalAdmin remain the currently implemented roles with duplicate-review permission. This test-only regression does not change role grants or establish product-owner approval of duplicate review responsibilities.


### Patient duplicate-review negative authorization — ClinicalUser (2026-11-27)

- API PR #118 merged as `56bfd4018b596834686eb00fb72d425dae10842d` after hosted API CI and Docker/PostgreSQL integration verification passed on head `42135b92862d4f537ef68aa5404f4b36d3ca66bd`.
- The PostgreSQL-backed regression verifies ClinicalUser receives HTTP 403 for duplicate-candidate review, the response does not expose submitted patient names, and the denied audit event retains tenant, validated issuer/subject and hospital resource attribution.
- Together with PR #117, FacilityManager and FacilityViewer denial coverage now exists for this operation. HospitalAdmin and PatientRegistrar remain the currently implemented grants. This does not approve those grants or complete the role/capability/scope matrix.


### Duplicate-review positive and hospital-scope regression — 2026-11-27

- API PR #119 merged as `bb3ab9868b58de247033b7a2695878c10e218902`: a PostgreSQL-backed test confirms HospitalAdmin can review duplicate candidates and receives the expected hospital-local patient candidate.
- API PR #120 merged as `25f4b300ce1bd85bb99a4db57f16affeee45f617`: a hospital-scoped HospitalAdmin receives the candidate within its assigned hospital but cannot query another hospital in the same tenant. The cross-hospital response is non-disclosing (404). The test verifies both the allowed and rejected audit events, including tenant, validated issuer/subject, action, resource ID/type, outcome and HTTP status.
- Both PRs are test-only and do not modify role grants. They prove implementation behavior, not product or clinical approval of duplicate-review responsibilities.

### Real Keycloak/API acceptance — duplicate review (2026-11-27)

- HeapCare.Local PR #30 merged as `981a63809db69861ec316d8bcb82a8d8ae5fee7`. The real-Keycloak acceptance script provisions synthetic data and verifies HospitalAdmin and PatientRegistrar can review duplicate candidates; FacilityManager, FacilityViewer and ClinicalUser are denied without submitted patient-name disclosure.
- The successful full-stack verification used isolated synthetic users and test data. It does not authorize production clinical use or replace the full endpoint/scope acceptance matrix.


### OPD queue department-scope regression — 2026-11-27

- **API PR #121 merged** as `9429d838c3c301f6b5434a29b25c4662aded6fc7`. PostgreSQL-backed coverage verifies a department-scoped ClinicalUser cannot list a different department's queue in the same hospital. The API returns a non-disclosing 404 and the denied audit event records the tenant, validated issuer/subject, `ListDepartmentQueue` action and status. No queue data is disclosed.
- **API PR #123 merged** as `9a4a517add68c5af08fc914533b387e6883440be`. A department-scoped PatientRegistrar can transition a queue entry in its assigned department, but receives a non-disclosing 404 for a queue entry in a peer department. The test verifies a successful `Called` transition and a `Waiting` queue item unchanged after the denied request. Both outcomes have separate audit events with validated tenant, issuer/subject, queue-entry resource ID, outcome and HTTP status.
- These tests validate resource-level organization-scope enforcement beyond the broad `queue.read` and `queue.manage` role grants. They do not alter the role resolver or approve broader queue responsibilities.
- API PR #122 also adds an audit CSV tenant-isolation regression: requesting-tenant actor metadata is included while another tenant's actor metadata is excluded. This is test evidence for tenant-bound export filtering, not approval of export purpose, retention or legal-hold policy.

The tests above are API/PostgreSQL regressions. The companion real Keycloak/Local acceptance remains a separate integration gate and must not be inferred from these component tests.

### Additional tenant/resource security and concurrency regressions — 2026-11-27

- API #122: PostgreSQL-backed audit CSV tenant isolation. Requesting-tenant actor metadata is included; another tenant's actor metadata is excluded.
- API #123: Department-scoped PatientRegistrar can transition an in-scope queue entry but cannot mutate a peer department's queue entry in the same hospital. The denial is non-disclosing (404), audited and leaves state unchanged.
- API #124: Eight simultaneous check-in requests for one booked appointment all return the same durable queue entry/token; PostgreSQL confirms a single appointment-linked row after the race. This tests the service's transaction-scoped advisory lock and recheck behavior.
- Local #33: real Keycloak tokens prove ClinicalUser cannot read cashier invoices or record payments and FacilityViewer cannot read invoices.
- Local #35: real Keycloak tokens prove availability publication is restricted to HospitalAdmin and availability read to PatientRegistrar; ClinicalUser cannot read availability and PatientRegistrar cannot create provider profiles.

These are synthetic isolated CI tests. They verify current implementation behavior, not product-owner approval of the role model, clinical privileges or minimum-necessary access. Production role sign-off, privacy/legal review and deployment-specific recovery remain separate gates.
### Clinical note lifecycle and confidentiality — Local #38 — 2026-11-27

- Local PR #38 merged as `fae841e12223e82ff6e7a8c0dfdf571222f744d6`. Real Keycloak-issued ClinicalUser can create an encounter from its assigned in-progress synthetic queue visit, save a note draft, sign it with persisted signer subject, close it and append an attributed amendment.
- A post-sign note overwrite returns HTTP 409. Re-reading after amendment confirms the original signed assessment is unchanged and amendment history contains one separately attributed item.
- PatientRegistrar receives HTTP 403 when attempting to read the clinical encounter. This verifies the separation between receptionist and clinical permissions with persisted memberships, not only unit-test principals.
- Existing PostgreSQL security coverage separately asserts clinical note text does not appear in audit metadata; note content is not placed in URLs, request identifiers or general audit fields.
- The Local real-OIDC test uses a dedicated synthetic CI patient and explicit placeholder note content in an ephemeral database only. It is not real patient data and does not approve clinical privilege or production use.


## Canonical catalogue and persisted RBAC implementation — 2026-11-28

**Review state:** The catalogue and persisted-RBAC implementation are available in open HeapCare.Api PRs #155 and #157. They have passed the current API CI, Docker and PostgreSQL migration checks, but are **not merged**. The current `main` branch remains the source of truth until the PRs merge in order.

### Canonical permission catalogue — PR #155

The Application-layer `TenantPermissionCatalog` defines the stable permission codes and their module, resource, action, display name, description and descriptive risk classification. Its system-role grant definitions preserve the exact baseline grants for:

- `HospitalAdmin`
- `FacilityManager`
- `FacilityViewer`
- `PatientRegistrar`
- `ClinicalUser`

All six existing IPD permissions are registered as non-assignable. No IPD role grant is introduced.

### Persisted tenant role assignments — PR #157

The proposed migration creates:

- `tenant_roles`: tenant-owned system/custom role definitions, stable role codes, active/system status and permission revision.
- `tenant_role_permissions`: tenant-owned role-to-permission assignments for code-owned permission identifiers.
- `tenant_role_change_audits`: append-only actor-attributed role changes with reason and previous/new name, description and permission snapshots.

The existing membership contract remains one role code per tenant membership. A composite foreign key links `tenant_memberships (TenantId, Role)` to `tenant_roles (TenantId, Code)`. The migration seeds the five system roles and backfills their exact existing grants for every tenant. Unknown legacy role codes are preserved as inactive, zero-permission roles; they are never mapped to a more privileged system role.

### Effective permission resolution and administration

The proposed runtime derives permission claims from the active persisted role and permission rows on each tenant-scoped request. Missing roles, inactive roles, empty grants, unknown permission identifiers and permissions invalid for the role type fail closed. Permission changes become effective on the next API request; neither token-supplied role/permission claims nor a process-local cache becomes the source of truth.

Tenant-wide membership administrators can create, edit, deactivate and reactivate custom role definitions. Role-definition changes require Tenant scope because the role definition is shared by the entire customer Tenant. Hospital- and department-scoped administrators cannot change a Tenant-wide role definition.

Custom role permissions must be a subset of the actor's own permissions. A separate protected permission, `membership.role.assign.system`, is held only by the immutable `HospitalAdmin` system role. It allows assignment of the five platform-approved system roles for compatibility with the existing membership workflow, but is itself non-assignable and cannot be delegated through a custom role. It is not available in the custom-role permission catalogue.

System roles cannot be edited, demoted or deactivated. Role rows cannot be physically deleted. Custom roles cannot be deactivated while active memberships still reference them. Role edits are serialized against membership role assignment; role-change history records actor, reason and before/after configuration and is protected against update/delete/truncate in PostgreSQL.

### Explicit limits

- A membership still has one role assignment. Multiple roles per membership are not implemented; a permission union across unrelated scopes is intentionally avoided.
- User-specific permission overrides are not implemented.
- Permission identifiers and metadata remain code-owned. Administrators can configure which known permissions a custom role receives without redeployment, but adding a new permission identifier still requires an API code change and release.
- Professional credentials and clinical privileges (licensure, specialty, prescribing/signing eligibility) remain separate from application RBAC.
- IPD permissions remain non-assignable and ungranted pending the separate IPD role/workflow approval.
- PlatformOperator remains a separate global identity; it receives no implicit tenant clinical access.

**Current branch caution:** until PR #155 merges, PR #157 merges after it, and the dependent Web PR #102 is accepted, the persisted role-management experience is not available from `main`.
