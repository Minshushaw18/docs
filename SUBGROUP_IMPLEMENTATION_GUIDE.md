# Subgroup Implementation Guide

## Overview
This document describes the complete implementation of the **Subgroup** feature for managing user progression through assessment-based workflows.

### Business Requirements
Users in a parent Group are organized into Subgroups based on their assessment performance:
- **Group 1 (INITIAL)**: Starting point for all users
- **Group 2/P1 (PASSED)**: Production group for users who pass assessments
- **Group 3/AS2 (RETRY_1)**: First retry assessment for users who fail
- **Group 4/AS3 (RETRY_2)**: Second retry assessment
- **Maximum 3 attempts** total before final placement

### Transition Rules
```
Group 1 → Group 2 (P1)    if score >= 80  [PASSED]
Group 1 → Group 3 (AS2)   if score < 80   [RETRY 1]
Group 3 → Group 2 (P1)    if score >= 80  [PASSED]
Group 3 → Group 4 (AS3)   if score < 80   [RETRY 2]
Group 4 → Group 2 (P1)    if score >= 80  [PASSED]
Group 4 → Group 4 (FINAL) if score < 80   [NO RETRY LEFT]
```

---

## Database Schema

### 1. Subgroup Model
**Location**: `label_studio/users/models.py`

**Purpose**: Represents a subgroup within a parent Group with assessment configuration

**Fields**:
- `name` (CharField): Display name (e.g., "Group 1", "Group 2 (P1)")
- `parent_group` (ForeignKey → Group): Parent group this belongs to
- `subgroup_type` (CharField): Type of subgroup
  - `INITIAL`: Initial assessment
  - `PASSED`: Passed - production
  - `RETRY_1`: First retry (AS2)
  - `RETRY_2`: Second retry (AS3)
  - `FINAL`: Final - no retry
- `description` (TextField): Optional description
- `assessment_project` (ForeignKey → Project): Assessment project for this stage
- `passing_score` (DecimalField): Minimum score to pass (default: 80.00)
- `next_subgroup_on_pass` (ForeignKey → self): Where to move user on passing
- `next_subgroup_on_fail` (ForeignKey → self): Where to move user on failing
- `max_retries` (PositiveIntegerField): Maximum attempts allowed (default: 3)
- `is_active` (BooleanField): Whether subgroup is active
- `created_by`, `created_at`, `updated_at`: Audit fields

**Relationships**:
- `projects` (ManyToMany → Project through SubgroupProject)
- `users` (ManyToMany → User through SubgroupMembership)

---

### 2. SubgroupMembership Model
**Purpose**: Tracks users in subgroups with their roles

**Fields**:
- `subgroup` (ForeignKey → Subgroup)
- `user` (ForeignKey → User)
- `role` (CharField): ANNOTATOR, REVIEWER, or ADMIN
- `joined_at` (DateTimeField): When user joined this subgroup
- `current_attempt` (PositiveIntegerField): Current attempt number
- `is_active` (BooleanField): Whether membership is active
- `exited_at` (DateTimeField): When user left this subgroup
- `created_at`, `updated_at`

**Unique Constraint**: (subgroup, user)

---

### 3. SubgroupProject Model
**Purpose**: Links projects to subgroups (defines project access)

**Fields**:
- `subgroup` (ForeignKey → Subgroup)
- `project` (ForeignKey → Project)
- `is_assessment` (BooleanField): Whether this is the assessment project
- `access_granted_at` (DateTimeField)
- `created_at`, `updated_at`

**Unique Constraint**: (subgroup, project)

---

### 4. UserSubgroupProgress Model
**Purpose**: Tracks each user's journey through the assessment flow

**Fields**:
- `user` (ForeignKey → User)
- `parent_group` (ForeignKey → Group)
- `current_subgroup` (ForeignKey → Subgroup): Current subgroup user is in
- `total_attempts` (PositiveIntegerField): Total assessment attempts made
- `latest_score` (DecimalField): Latest score achieved
- `latest_assessment_date` (DateTimeField): Date of latest assessment
- `status` (CharField):
  - `IN_PROGRESS`: Currently working through assessments
  - `PASSED`: Successfully passed and moved to production
  - `FAILED`: Failed but can retry
  - `NO_RETRY`: No retries left
- `transition_history` (JSONField): Array of all transitions with scores

**Unique Constraint**: (user, parent_group)

**Methods**:
- `add_transition(from_subgroup, to_subgroup, score, passed)`: Record transition
- `can_retry()`: Check if user has attempts remaining

---

## Service Layer

### SubgroupTransitionService
**Location**: `label_studio/users/services/subgroup_service.py`

**Key Methods**:

#### 1. `calculate_assessment_score(user, project)`
Calculates assessment score for a user:
```python
score = (golden_benchmarks_accepted / golden_benchmarks_attempted) * 100
```
Returns `Decimal` score or `None` if no attempts.

#### 2. `process_assessment_result(user, assessment_project)`
Main workflow method:
1. Calculate user's score
2. Determine if passed (score >= passing_score)
3. Check retry eligibility
4. Move user to next subgroup
5. Update progress tracking

Returns: `(success: bool, message: str, next_subgroup: Subgroup|None)`

#### 3. `move_user_to_subgroup(user, from_subgroup, to_subgroup, score, passed)`
- Deactivates old subgroup membership
- Creates/activates new subgroup membership
- Updates progress tracking
- Records transition history

#### 4. `initialize_user_in_group(user, parent_group, initial_subgroup_type)`
Places a new user into the initial subgroup (typically Group 1).

#### 5. `get_user_accessible_projects(user, parent_group)`
Returns list of project IDs user can access based on their current subgroup.

#### 6. `get_user_progress_summary(user, parent_group)`
Returns detailed progress information including:
- Current subgroup
- Total attempts
- Latest score
- Retry eligibility
- Transition history

---

## API Endpoints

### Base URL: `/api/user-admin/subgroups/`

### 1. List/Create Subgroups
**GET** `/api/user-admin/subgroups/`
- Query params: `parent_group` (optional)
- Returns: List of subgroups

**POST** `/api/user-admin/subgroups/`
- Body:
  ```json
  {
    "name": "Group 1",
    "parent_group": 1,
    "subgroup_type": "INITIAL",
    "description": "Initial assessment group",
    "assessment_project": 331,
    "passing_score": 80.0,
    "next_subgroup_on_pass": 2,
    "next_subgroup_on_fail": 3,
    "max_retries": 3,
    "project_ids": [331, 332]
  }
  ```

### 2. Subgroup Detail
**GET** `/api/user-admin/subgroups/<id>/`
- Returns: Detailed subgroup info with members and projects

**PATCH** `/api/user-admin/subgroups/<id>/`
- Update subgroup fields
- Body: Partial update of any field

**DELETE** `/api/user-admin/subgroups/<id>/`
- Deletes subgroup

### 3. Add Users to Subgroup
**POST** `/api/user-admin/subgroups/<id>/add-users/`
```json
{
  "user_ids": [1, 2, 3],
  "role": "ANNOTATOR"
}
```

### 4. Remove Users from Subgroup
**POST** `/api/user-admin/subgroups/<id>/remove-users/`
```json
{
  "user_ids": [1, 2, 3]
}
```

### 5. Get User Progress
**GET** `/api/user-admin/subgroups/progress/`
- Query params: `user_id` OR `group_id`
- Returns: Progress records with scores, attempts, history

### 6. Process Assessment
**POST** `/api/user-admin/subgroups/process-assessment/`
```json
{
  "user_id": 123,
  "project_id": 331
}
```
- Calculates score from completed assessment
- Transitions user to appropriate subgroup
- Returns: Success status, message, next subgroup info

### 7. Initialize Users in Group
**POST** `/api/user-admin/subgroups/initialize-users/`
```json
{
  "group_id": 1,
  "user_ids": [1, 2, 3],
  "initial_subgroup_type": "INITIAL"
}
```
- Places users into the initial subgroup of a group

---

## Serializers

**Location**: `label_studio/user_admin/serializers.py`

### Key Serializers:
1. **SubgroupListSerializer**: List view with counts
2. **SubgroupDetailSerializer**: Full details with nested members/projects
3. **SubgroupCreateSerializer**: Create with project associations
4. **SubgroupUpdateSerializer**: Update with add/remove projects
5. **UserSubgroupProgressSerializer**: Progress tracking details
6. **ProcessAssessmentSerializer**: Validation for assessment processing

---

## Setup Instructions

### Step 1: Run Migrations
```bash
python label_studio/manage.py makemigrations users
python label_studio/manage.py migrate users
```

### Step 2: Create Parent Group
Via Django admin or API:
```python
group = Group.objects.create(
    name="English IPA",
    workspace=workspace,
    created_by=admin_user
)
```

### Step 3: Create Subgroups with Transitions

**Group 1 (Initial)**:
```python
group1 = Subgroup.objects.create(
    name="Group 1",
    parent_group=group,
    subgroup_type="INITIAL",
    assessment_project=assessment_project_1,
    passing_score=80.0,
    max_retries=3,
    created_by=admin_user
)
```

**Group 2 (Passed/Production)**:
```python
group2 = Subgroup.objects.create(
    name="Group 2 (P1)",
    parent_group=group,
    subgroup_type="PASSED",
    passing_score=80.0,
    created_by=admin_user
)
```

**Group 3 (Retry 1)**:
```python
group3 = Subgroup.objects.create(
    name="Group 3 (AS2)",
    parent_group=group,
    subgroup_type="RETRY_1",
    assessment_project=assessment_project_2,
    passing_score=80.0,
    max_retries=3,
    created_by=admin_user
)
```

**Group 4 (Retry 2)**:
```python
group4 = Subgroup.objects.create(
    name="Group 4 (AS3)",
    parent_group=group,
    subgroup_type="RETRY_2",
    assessment_project=assessment_project_3,
    passing_score=80.0,
    max_retries=3,
    created_by=admin_user
)
```

### Step 4: Configure Transitions
```python
# Group 1 transitions
group1.next_subgroup_on_pass = group2
group1.next_subgroup_on_fail = group3
group1.save()

# Group 3 transitions
group3.next_subgroup_on_pass = group2
group3.next_subgroup_on_fail = group4
group3.save()

# Group 4 transitions
group4.next_subgroup_on_pass = group2
group4.next_subgroup_on_fail = None  # No more retries
group4.save()
```

### Step 5: Associate Projects
```python
# Add projects to each subgroup
SubgroupProject.objects.create(subgroup=group1, project=assessment_project_1, is_assessment=True)
SubgroupProject.objects.create(subgroup=group2, project=production_project_1, is_assessment=False)
# ... etc
```

### Step 6: Initialize Users
```python
from users.services import SubgroupTransitionService

for user in users_to_enroll:
    SubgroupTransitionService.initialize_user_in_group(
        user=user,
        parent_group=group,
        initial_subgroup_type='INITIAL'
    )
```

---

## Usage Example: Complete Flow

### 1. User Starts in Group 1
```python
# User is initialized
SubgroupTransitionService.initialize_user_in_group(
    user=user,
    parent_group=english_ipa_group,
    initial_subgroup_type='INITIAL'
)
```

### 2. User Completes Assessment
User works on assessment project and completes golden benchmark tasks.

### 3. Process Assessment Result
```python
success, message, next_subgroup = SubgroupTransitionService.process_assessment_result(
    user=user,
    assessment_project=assessment_project_1
)
```

**If score >= 80**:
- User moves to Group 2 (P1)
- Status: PASSED
- Can access production projects

**If score < 80**:
- User moves to Group 3 (AS2)
- Total attempts: 2
- Can retry assessment

### 4. Check User Progress
```python
progress = SubgroupTransitionService.get_user_progress_summary(
    user=user,
    parent_group=english_ipa_group
)

# Returns:
{
    'current_subgroup': 'Group 3 (AS2)',
    'subgroup_type': 'Retry 1 - AS2',
    'total_attempts': 2,
    'latest_score': 75.0,
    'status': 'In Progress',
    'can_retry': True,
    'attempts_left': 1,
    'transition_history': [
        {
            'from_subgroup': None,
            'to_subgroup': 'Group 1',
            'score': None,
            'passed': False,
            'attempt': 1,
            'timestamp': '2025-01-15T10:00:00Z'
        },
        {
            'from_subgroup': 'Group 1',
            'to_subgroup': 'Group 3 (AS2)',
            'score': 75.0,
            'passed': False,
            'attempt': 2,
            'timestamp': '2025-01-16T14:30:00Z'
        }
    ]
}
```

### 5. User Retries and Passes
After completing second assessment with score >= 80:
- Moves to Group 2 (P1)
- Status: PASSED
- Gets access to production projects

---

## URL Configuration

Add to `label_studio/user_admin/urls.py`:
```python
from .api import (
    SubgroupListCreateAPIView,
    SubgroupDetailAPIView,
    SubgroupAddUsersAPIView,
    SubgroupRemoveUsersAPIView,
    UserSubgroupProgressAPIView,
    ProcessAssessmentAPIView,
    InitializeUserInGroupAPIView,
)

urlpatterns = [
    # ... existing patterns ...

    # Subgroup endpoints
    path('subgroups/', SubgroupListCreateAPIView.as_view(), name='subgroup-list-create'),
    path('subgroups/<int:pk>/', SubgroupDetailAPIView.as_view(), name='subgroup-detail'),
    path('subgroups/<int:pk>/add-users/', SubgroupAddUsersAPIView.as_view(), name='subgroup-add-users'),
    path('subgroups/<int:pk>/remove-users/', SubgroupRemoveUsersAPIView.as_view(), name='subgroup-remove-users'),
    path('subgroups/progress/', UserSubgroupProgressAPIView.as_view(), name='subgroup-progress'),
    path('subgroups/process-assessment/', ProcessAssessmentAPIView.as_view(), name='process-assessment'),
    path('subgroups/initialize-users/', InitializeUserInGroupAPIView.as_view(), name='initialize-users'),
]
```

---

## Assessment Score Calculation

The assessment score is calculated from golden benchmark tasks:

```python
def calculate_assessment_score(user, project):
    """
    Score = (Accepted Golden Benchmarks / Total Golden Benchmarks Attempted) * 100
    """
    annotations = Annotation.objects.filter(
        project=project,
        completed_by=user.id,
        was_cancelled=False
    )

    attempted = annotations.filter(
        task__task_status=TaskStatus.GOLDEN_REFERRENCE
    ).count()

    accepted = annotations.filter(
        review_status=ReviewStatus.ACCEPTED,
        task__task_status=TaskStatus.GOLDEN_REFERRENCE
    ).count()

    if attempted > 0:
        return Decimal(100) * Decimal(accepted) / Decimal(attempted)

    return None
```

---

## Key Features

### 1. Automatic Transitions
Users are automatically moved between subgroups based on assessment performance.

### 2. Retry Tracking
System tracks total attempts and enforces maximum retry limit (3 attempts).

### 3. Progress History
Complete audit trail of all transitions with scores and timestamps.

### 4. Project Access Control
Users only see projects associated with their current subgroup.

### 5. Flexible Configuration
- Configurable passing scores per subgroup
- Customizable max retries
- Multiple assessment projects support

### 6. Production-Ready
- Transaction safety with `@transaction.atomic`
- Comprehensive error handling
- Logging for debugging
- Optimized queries with select_related/prefetch_related

---

## Testing Checklist

- [ ] Create parent group
- [ ] Create 4 subgroups (Group 1, 2, 3, 4)
- [ ] Configure transitions between subgroups
- [ ] Associate assessment projects
- [ ] Initialize users in Group 1
- [ ] User completes assessment with score >= 80 → moves to Group 2
- [ ] User completes assessment with score < 80 → moves to Group 3
- [ ] User in Group 3 passes → moves to Group 2
- [ ] User in Group 3 fails → moves to Group 4
- [ ] User in Group 4 fails → stays in Group 4 (no retry)
- [ ] Check progress API returns correct history
- [ ] Verify project access restrictions

---

## Future Enhancements

1. **Email Notifications**: Send emails on transitions
2. **Admin Dashboard**: Visual flow chart of subgroup transitions
3. **Bulk Operations**: Process assessments for multiple users
4. **Custom Rules**: Support for more complex transition logic
5. **Analytics**: Reports on pass rates, average scores, etc.

---

## Troubleshooting

### Issue: User not transitioning after assessment
**Check**:
- Assessment project has `assessment=True` flag
- Golden benchmark tasks exist and are marked correctly
- User has completed annotations that are ACCEPTED
- Subgroup transitions are configured (`next_subgroup_on_pass/fail`)

### Issue: Score calculation returns None
**Check**:
- User has attempted golden benchmark tasks
- Annotations are not cancelled
- Task status is GOLDEN_REFERRENCE

### Issue: User can retry more than 3 times
**Check**:
- `max_retries` is set correctly on subgroups
- `can_retry()` method is being called before processing
- Progress `total_attempts` is being incremented

---

## Contact & Support

For questions or issues:
- Review the code in `label_studio/users/models.py`
- Check service logic in `label_studio/users/services/subgroup_service.py`
- Examine API endpoints in `label_studio/user_admin/api.py`

---

**Implementation Date**: January 2025
**Version**: 1.0
**Status**: Production Ready (pending migrations)
