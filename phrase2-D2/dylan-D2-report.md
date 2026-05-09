The core data model is designed to support the complete lifecycle of a research ethics application, ensuring consistency. Our entity design addresses three key architectural priorities: process automation, compliance Auditing, and state consistency.

## Core Data Model Entities & Architectural Mapping

| Class Name              | Component Owner (D1)  | Key Attributes                                                       | Description & Requirement Mapping                                                  |
| :---------------------- | :-------------------- | :------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| **User**                | Access Control Module | `user_id`, `name`, `email`, `role`                                   | System actors; role-based access enforcement. **(REQ-AC-01)**                      |
| **Application**         | Workflow Module       | `app_id`, `parent_app_id`, `title`, `app_type`, `status`, `deadline` | Lifecycle management; recursive linking for amendments. **(REQ-WF-01, REQ-WF-04)** |
| **Assignment**          | Workflow Module       | `assign_id`, `app_id`, `member_id`, `status`                         | Task delegation & reviewer mapping. **(REQ-WF-02)**                                |
| **ReviewDecision**      | Review Module         | `decision_id`, `assign_id`, `type`, `is_submitted`                   | Formal reviewer response; triggers re-review loop. **(REQ-REV-01)**                |
| **Annotation**          | Review Module         | `annot_id`, `decision_id`, `version_id`, `content`                   | Section-specific feedback linked to a doc version. **(REQ-REV-04)**                |
| **ConflictDeclaration** | Access Control Module | `decl_id`, `assign_id`, `member_id`, `reason`                        | Formal COI recording; blocks assignment logic. **(REQ-AC-03)**                     |
| **Document**            | Review Module         | `doc_id`, `app_id`, `doc_type`                                       | Logical container for research documents. **(REQ-DOC-01)**                         |
| **DocumentVersion**     | Review Module         | `version_id`, `doc_id`, `number`, `path`, `is_current`               | Binary metadata & version history tracking. **(REQ-DOC-01, REQ-US-01)**            |
| **Communication**       | Notification Module   | `comm_id`, `app_id`, `sender_id`, `receiver_id`, `type`              | Admin-to-PI formal revision requests. **(REQ-REV-03)**                             |
| **AuditEntry**          | Workflow Module       | `audit_id`, `app_id`, `actor_id`, `action`, `timestamp`              | Full operational trail for compliance auditing. **(REQ-REV-03)**                   |


## **Class Diagram**
![[dylan-d2.png]]

## Relationships and Cardinalities

The following associations define the structural constraints of the REMS system:
- User to Application (`1:0..*`): A PI initiates multiple applications; each application is owned by exactly one PI, establishing primary accountability.
- User to Assignment (`1:0..*`): An Ethics Committee Member (User) receives multiple assignment tasks. **(REQ-WF-02)**
- User to AuditEntry (`1:0..*`): Every action in the system is performed by a User. This ensures all state-changing operations are attributable to a specific actor for compliance. **(REQ-AC-03)**
- User to Communication (`1:0..*`): Administrative Staff (User) act as the Sender for formal communications, ensuring the source of every revision request is identified. **(REQ-REV-03)**
- Application to Application (Recursive: `0..1:0..*`): Enables Amendment and Extension flows by linking new applications to their predecessors, ensuring a clear version lineage. **(REQ-WF-01)**
- Application to Assignment (`1:0..*`): Links an application to reviewer assignments for collaborative review.
- Assignment to ReviewDecision (`1:0..1`): Each assignment task maps to at most one formal decision. A null decision represents a pending review state. **(REQ-WF-03)**
- ReviewDecision to Annotation (`1:0..*`): A reviewer’s decision can contain multiple section-specific annotations. **(REQ-REV-04)**
- Assignment to ConflictDeclaration (`1:0..1`):  Tracks COI declarations linked to a specific reviewer assignment; an unresolved declaration (status = 'Pending') serves as a logic gate that blocks the submission of a ReviewDecision for that assignment. **(REQ-AC-03)**
- Application to Document (`1:1..*`): Every application contains a set of logical documents; documents are strictly scoped to the application.
- Document to DocumentVersion (`1:1..*`): Documents evolve through multiple versions; the system tracks these for side-by-side comparison. **(REQ-DOC-03)**
- Application to Communication (`1:0..*`): Stores all formal correspondence to provide a structured revision history. **(REQ-REV-03)**
- Application to AuditEntry (`1:0..*`): Records the complete operational log of the application lifecycle, essential for regulatory compliance.

## Design Constraints

- **Constraint 1: Conflict-of-Interest Blocking**
    - Rule: A ReviewDecision cannot be submitted if there exists a ConflictDeclaration for the same Assignment with a status of Pending.
    - Purpose: Ensures ethical integrity by preventing conflicted members from influencing the review process.
- **Constraint 2: Revision Loop Consistency**
    - Rule: Upon the upload of a new DocumentVersion (where IsCurrent = true), the system must automatically reset the IsSubmitted flag of the associated ReviewDecision to false.
    - Purpose: Forces a re-review loop, ensuring committee feedback is always based on the most recent document content.
- **Constraint 3: Administrative Buffer (Traceability)**
    - Rule: Every Communication of type RevisionRequest must be logically traceable to a ReviewDecision via the audit log or system linkage.
    - Purpose: Enforces the administrative buffer, ensuring all requests to researchers are grounded in committee feedback.
- **Constraint 4: Version Immutability**
    - Rule: A DocumentVersion record, once created, cannot be modified. Only a new version can be created.
    - Purpose: Provides the "Single Source of Truth" required for historical auditing.
- **Constraint 5: Operational Immutability**
	- **Rule**: Records in the AuditEntry and DocumentVersion entities are immutable; they cannot be updated or deleted after creation.
    - **Purpose**: Ensures the integrity of the compliance trail. If a record is deemed incorrect, a new "correction" record must be created in the AuditEntry log rather than modifying the original, satisfying the requirement for an "auditable mechanism" (Phase 1).
- **Constraint 6: Conflict-of-Interest Lifecycle**
	- **Rule**: An Assignment can have at most one ConflictDeclaration in a Pending status.
	- **Purpose**: Prevents the system from being cluttered with redundant conflict filings and ensures that Chair interventions (reassignment) are processed linearly and decisively.
