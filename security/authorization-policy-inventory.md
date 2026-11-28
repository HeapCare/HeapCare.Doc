# API authorization policy inventory

**Status:** Source-derived inventory, updated 2026-11-28; persisted role administration is under review in HeapCare.Api PR #157 and is not yet merged.  
**Purpose:** Provide a traceable starting point for role-based authorization review and authenticated acceptance. This is not a role-to-permission grant matrix and is not a substitute for tests or a live authorization review.

## Source of truth and trust model

The API defines named authorization policies in `src/HeapCare.Api/Security/AuthorizationPolicies.cs`. Tenant permission policies require an authenticated principal and a permission claim on the dedicated `HeapCareTenantMembership` identity. The API creates that identity after resolving the validated OIDC issuer/subject against an active persisted tenant membership and deriving permissions from the persisted role. Token-supplied role and permission claims are not the source of authorization.

Every tenant-scoped route group must retain its tenant-scoped endpoint metadata and authenticated requirement. Individual operations then apply the narrow policy listed below. Platform-operator operations use separate platform-operator metadata and policy. The `X-Tenant-ID` header selects context; it never grants access.

## Policy inventory

| Capability | API policy | Applied operations / source endpoint |
|---|---|---|
| Facility read | `FacilityRead` | Hospital list/detail and corresponding hierarchy read operations; `FacilityEndpoints.cs` and sibling hierarchy endpoint groups |
| Facility create/update | `FacilityWrite` | Hospital create/update and corresponding hierarchy writes; `FacilityEndpoints.cs` and sibling hierarchy endpoint groups |
| Facility delete | `FacilityDelete` | Hospital delete and corresponding hierarchy deletes; `FacilityEndpoints.cs` and sibling hierarchy endpoint groups |
| Tenant membership administration | `TenantMembershipManage` | Tenant membership management group; `TenantMembershipEndpoints.cs` |
| Tenant role administration | `TenantMembershipManage` | Tenant role list, delegable permission catalogue, custom role create/update/deactivate/reactivate; `TenantRoleEndpoints.cs`. Service layer restricts role-definition changes to Tenant scope and validates permission delegation. |
| Platform operator | `PlatformOperator` | Initial-administrator provisioning and platform-operator operations; `TenantMembershipEndpoints.cs` and platform endpoint groups |
| Patient read | `PatientRead` | Patient search/read operations; `PatientEndpoints.cs` |
| Patient identity read | `PatientIdentityRead` | Patient identity/detail operations; `PatientEndpoints.cs` |
| Patient registration | `PatientRegister` | Patient registration; `PatientEndpoints.cs` |
| Duplicate candidate review | `PatientDuplicateReview` | Duplicate candidate review; `PatientEndpoints.cs` |
| Patient identifier read | `PatientIdentifierRead` | Identifier lookup/read operations; `PatientIdentifierEndpoints.cs` |
| Patient identifier assignment | `PatientIdentifierAssign` | Identifier assignment; `PatientIdentifierEndpoints.cs` |
| Identifier policy read/manage | `PatientIdentifierPolicyRead` / `PatientIdentifierPolicyManage` | Hospital MRN/identifier policy operations; `PatientIdentifierEndpoints.cs` |
| Staff read/manage | `StaffRead` / `StaffManage` | Staff profile and assignment read/write/deactivate/reactivate; `StaffEndpoints.cs` |
| Availability read/manage | `AvailabilityRead` / `AvailabilityManage` | Appointment availability reads and changes; `AppointmentEndpoints.cs` |
| Appointment read/manage | `AppointmentRead` / `AppointmentManage` | Appointment read, booking, cancellation, reschedule and check-in operations; `AppointmentEndpoints.cs` |
| Queue read/manage | `QueueRead` / `QueueManage` | Queue listing and queue/walk-in transitions; `AppointmentEndpoints.cs` |
| Encounter read/manage | `EncounterRead` / `EncounterManage` | OPD encounter read and draft/update workflow; `OpdEncounterEndpoints.cs` |
| Encounter sign/amend | `EncounterSign` / `EncounterAmend` | Clinical signing and amendment operations; `OpdEncounterEndpoints.cs` |
| Billing catalogue | `BillingCatalogueManage` | Service-item create/update; `BillingEndpoints.cs` |
| Billing invoice read/manage | `BillingInvoiceRead` / `BillingInvoiceManage` | Invoice read and creation; `BillingEndpoints.cs` |
| Billing payment | `BillingPaymentManage` | Payment recording; `BillingEndpoints.cs` |
| Billing reports | `BillingReportRead` | Daily collection report; `BillingEndpoints.cs` |
| Audit read | `AuditRead` | Audit event query/export operations; `AuditEndpoints.cs` |

## Important scope distinctions

- The policy names above are taken from the API policy constants and endpoint registrations. They describe authorization checks at the HTTP boundary; they do not by themselves prove resource-level hospital/department scope checks inside application services or persistence queries.
- The route group itself may add authentication and tenant/platform metadata inherited by each endpoint. Reviews must inspect inherited metadata as well as per-operation policies.
- The current persisted tenant-role grants are documented separately in the [persisted role-to-permission matrix](persisted-role-permission-matrix.md). That matrix is a source-derived implementation snapshot, not product-owner approval or proof of endpoint/resource-scope correctness. The API currently has five tenant roles; `PlatformOperator` is a separate global identity/policy and must be reviewed independently. Do not infer a role grant from a policy name.
- Billing invoice read is also used to list catalogue items in the current endpoint registration. Confirm this remains intentional when the billing permission model is revised; do not silently assume a separate catalogue-read permission exists.
- The staff module README says the initial staff read/manage permissions are granted only to HospitalAdmin. Preserve this restriction unless a reviewed role matrix and negative authorization tests explicitly approve a change.
- UI navigation and component visibility are usability controls only. Every API operation must independently enforce its policy and resource/organization scope.

## Required regression matrix

For every capability above, maintain positive and negative integration coverage using signed synthetic OIDC identities and persisted memberships. The test matrix must vary:

1. No bearer token, invalid signature, wrong issuer, wrong audience, expired token and missing subject.
2. No tenant header, malformed/multiple tenant headers, inactive tenant and no matching membership.
3. Each supported persisted role with only its intended permission set; a token forged with extra legacy role/permission claims must not change the outcome.
4. Tenant-wide, hospital-scoped and department-scoped memberships, including cross-tenant, cross-hospital and same-hospital cross-department resource IDs.
5. Read versus create/update versus delete; draft versus sign versus amend; invoice read versus invoice create versus payment recording; audit query versus export.
6. Denied requests must not mutate business state. Sensitive denied operations must emit the approved metadata-only denial audit event where the route is configured to do so.
7. Every authenticated route must be discovered by the endpoint audit-metadata guard; tests must fail if route discovery returns zero endpoints.

## Verification references

- API `src/HeapCare.Api/Security/AuthorizationPolicies.cs` — policy constants and membership-identity permission evaluation.
- API `src/HeapCare.Api/Endpoints/FacilityEndpoints.cs` — facility read/write/delete operation policy assignments.
- API `src/HeapCare.Api/Endpoints/PatientEndpoints.cs` — patient registration, duplicate review and read/identity-read assignments.
- API `src/HeapCare.Api/Endpoints/StaffEndpoints.cs` — staff read/manage assignments.
- API `src/HeapCare.Api/Endpoints/AppointmentEndpoints.cs` — availability, appointment and queue assignments.
- API `src/HeapCare.Api/Endpoints/OpdEncounterEndpoints.cs` — encounter read/manage/sign/amend assignments.
- API `src/HeapCare.Api/Endpoints/BillingEndpoints.cs` — catalogue, invoice, payment and report assignments.
- API `src/HeapCare.Api/Endpoints/AuditEndpoints.cs` — audit read assignments.
- API `src/HeapCare.Api/Endpoints/TenantMembershipEndpoints.cs` — membership and platform-operator boundary.
- API `README.md` — documented OIDC, tenant membership, facility and staff security behavior.

## Release gate

Do not treat this inventory as proof of role correctness. Before production/pilot approval, reconcile the persisted role-to-permission resolver with the product role matrix, inspect every endpoint and service-level scope check, execute the complete positive/negative PostgreSQL integration matrix, and repeat authenticated local Keycloak/API/Web acceptance. Any permission expansion requires a reviewed change, explicit negative tests and updated user/developer documentation.

## Latest authorization regression evidence — 2026-11-27

The source-derived role matrix now records the merged API PRs #96–#100. In brief: PR #96 covers hospital/department-scoped patient identifier denial; PR #97 covers OPD encounter denials for FacilityManager, FacilityViewer and PatientRegistrar; PR #98 adds FacilityManager scheduling/billing denials and patient search/registration denials for FacilityManager/FacilityViewer; PR #99 explicitly denies HospitalAdmin OPD encounter access; and PR #100 verifies hospital/department-scoped HospitalAdmin cannot query/export tenant-wide audit data. These are targeted regression tests, not an exhaustive authorization certification. The next review must compare every current endpoint registration and service/persistence scope guard with the latest HeapCare.Api source and preserve the distinction between policy-level and resource-level authorization.


## Persisted role-administration routes — PR #157 (not yet merged)

The proposed tenant role API is registered under the existing authenticated tenant-scoped route metadata and the `TenantMembershipManage` policy:

| Operation | Route | Endpoint name |
|---|---|---|
| List roles visible to the actor | `GET /api/v1/tenant/roles` | `ListTenantRoles` |
| List permission definitions the actor may delegate | `GET /api/v1/tenant/roles/permissions` | `ListAssignableTenantRolePermissions` |
| Create a custom role | `POST /api/v1/tenant/roles` | `CreateTenantRole` |
| Update a custom role | `PUT /api/v1/tenant/roles/{roleId}` | `UpdateTenantRole` |
| Deactivate a custom role | `POST /api/v1/tenant/roles/{roleId}/deactivate` | `DeactivateTenantRole` |
| Reactivate a custom role | `POST /api/v1/tenant/roles/{roleId}/reactivate` | `ReactivateTenantRole` |

The endpoint policy is only the first authorization layer. The administration service additionally requires Tenant scope for role-definition changes, rejects non-assignable or unknown permission codes, enforces custom-role grant-subset delegation, protects system roles and appends a before/after audit snapshot with a required reason. These routes are part of open PR #157, not the current `main` endpoint surface.
