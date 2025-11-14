#  PROJECTS & CONFIGURATION - DATABASE SCHEMA DOCUMENTATION
## Complete Column Use Cases & Business Logic

**Responsibility:** Project setup, configuration, tags, data views, and webhooks
**Sections:** 8, 9, 10, 17, 19, 20
**Total Tables:** 21 tables
**Complexity:** High

---

## 📋 TABLE OF CONTENTS

### Section 8: Core Projects
1. [project](#1-project)
2. [ProjectSummary](#2-projectsummary)
3. [ProjectNestedRelations](#3-projectnestedrelations)

### Section 9: Project Configuration & Management
4. [projects_tag](#4-projects_tag)
5. [ProjectTag](#5-projecttag)
6. [ProjectClientApiConfiguration](#6-projectclientapiconfiguration)
7. [ProjectOnboardingSteps](#7-projectonboardingsteps)
8. [ProjectOnboarding](#8-projectonboarding)
9. [ProjectMember](#9-projectmember)
10. [ProjectGroup](#10-projectgroup)
11. [ProjectOpsManager](#11-projectopsmanager)
12. [ProjectEmbedding](#12-projectembedding)
13. [TemplateDashboard](#13-templatedashboard)

### Section 10: Project Data Import/Export
14. [ProjectImport](#14-projectimport)
15. [ProjectReimport](#15-projectreimport)
16. [FileUpload](#16-fileupload)

### Section 17: Labels & Tags
17. [Label](#17-label)
18. [LabelLink](#18-labellink)

### Section 19: Data Manager & Views
19. [data_manager_view](#19-data_manager_view)
20. [FilterGroup](#20-filtergroup)
21. [Filter](#21-filter)

### Section 20: Webhooks
22. [webhook](#22-webhook)
23. [webhook_action](#23-webhook_action)

---

## 🎯 KEY USE CASES FOR PROJECTS & CONFIGURATION

### High-Level Business Flows:
1. **Project Creation & Setup** - Creating annotation projects with labeling interfaces and instructions
2. **Label Configuration Management** - XML-based UI configuration for annotation interfaces
3. **Task Import & Data Ingestion** - Bulk importing tasks from CSV, JSON, cloud storage
4. **Project Member Management** - Assigning annotators, reviewers, and admins to projects
5. **Quality Control Configuration** - Setting up overlap, review percentages, golden reference tasks
6. **ML Model Integration** - Connecting ML backends for predictions and active learning
7. **Data Export & Format Conversion** - Exporting annotations in multiple formats (JSON, CSV, COCO, YOLO)
8. **Project Tagging & Organization** - Categorizing projects with tags for filtering
9. **Webhook Integration** - Real-time event notifications to external systems
10. **Custom Dashboard Embedding** - Embedding analytics dashboards in project UI
11. **Data Manager Views** - Saving custom filters and column configurations
12. **Project Nesting** - Parent-child project relationships for multi-stage workflows
13. **Task Distribution Strategies** - Configuring how tasks are assigned (equal distribution, max limits, custom rules)
14. **Label Library Management** - Reusable label definitions across projects

---

## SECTION 8: CORE PROJECTS

---

## 1. project

**Purpose:** Core annotation project containing all configuration for labeling workflows
**Table Name:** `project`
**Business Context:** Central entity for data annotation. Defines what data to annotate, how to annotate it, who can annotate, and quality control rules

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Project ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for project. Referenced by tasks, annotations, imports, exports |
| **title** | varchar | Project name | NULL, VALIDATED (min/max length) | **Project identification** - Display name in UI (e.g., "Medical Image Classification Q1 2024"). Validates between min/max chars (e.g., 3-100). Used in project lists, breadcrumbs, notifications |
| **description** | text | Project description | NULL | **Project documentation** - Detailed explanation of project goals, data types, expected outcomes. Helps team understand project purpose months later. Shown in project overview |
| **project_type** | varchar(100) | Type of project | CHOICES: DP_ASSESSMENT/DP_ORDER/LABELLING/ASSESSMENT/DOUBLE_LABELLING, NULL | **Workflow determination** - Controls which features available:<br>- **DP_ASSESSMENT:** Data production assessment workflow<br>- **DP_ORDER:** Commissioned data collection orders<br>- **LABELLING:** Standard annotation project<br>- **ASSESSMENT:** Quality assessment workflow<br>- **DOUBLE_LABELLING:** Requires 2+ annotations per task for agreement |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE, NULL | **CRITICAL: Workspace scoping** - Project belongs to workspace. All project data isolated by workspace. CASCADE delete removes project when workspace deleted |
| **label_config** | text | XML labeling interface | DEFAULT '<View></View>', NULL | **MOST CRITICAL: UI definition** - XML configuration defining annotation interface:<br>- What UI controls shown (bounding boxes, text input, choices)<br>- What data fields to annotate<br>- Validation rules<br>- Hotkeys<br>Example: `<View><Image name="img" value="$image"/><Choices name="sentiment"><Choice value="positive"/></Choices></View>` |
| **parsed_label_config** | jsonb | Parsed JSON label config | NULL | **Performance cache** - Pre-parsed JSON version of label_config XML. Auto-generated on save via parse_config(). Avoids parsing XML on every task load. Invalidated when label_config changes |
| **label_config_hash** | bigint | Hash of label config | NULL | **Change detection** - Hash of parsed_label_config. Quick check if config changed without comparing full JSON. Used to trigger cache invalidation |
| **assessment** | boolean | Assessment workflow flag | DEFAULT FALSE | **Workflow toggle** - TRUE for assessment-type projects (quality evaluation). Changes available actions and UI |
| **expert_instruction** | text | Instructions for annotators | DEFAULT '', NULL | **Annotator guidance** - HTML instructions shown to annotators. Sanitized with bleach.clean() to prevent XSS. Can include:<br>- Task guidelines<br>- Examples of good/bad annotations<br>- Edge case handling<br>- Images, links, formatting |
| **show_instruction** | boolean | Show instructions toggle | DEFAULT FALSE | **UI control** - TRUE: Show expert_instruction before annotation. FALSE: Hide instructions (accessible via help button). Used to reduce screen clutter |
| **show_skip_button** | boolean | Enable skip button | DEFAULT TRUE | **Task progression** - TRUE: Annotators can skip difficult tasks. FALSE: Must annotate every task. Affects task completion logic (skip_queue setting) |
| **enable_empty_annotation** | boolean | Allow empty submissions | DEFAULT TRUE | **Quality control** - TRUE: Allow submitting without annotations (for "nothing to annotate" cases). FALSE: Require at least one annotation. Validated in AnnotationSerializer |
| **reveal_preannotations_interactively** | boolean | Interactive pre-annotations | DEFAULT FALSE | **ML UX** - TRUE: Pre-annotations revealed one-by-one as annotator creates regions. FALSE: All pre-annotations shown immediately. Affects cognitive load |
| **show_annotation_history** | boolean | Show annotation history | DEFAULT FALSE | **Review mode** - TRUE: Annotators see previous annotations on task. FALSE: Blind annotation. Used for double-labeling and review workflows |
| **show_collab_predictions** | boolean | Show ML predictions | DEFAULT TRUE | **ML integration** - TRUE: Display predictions from ML backends to annotators. FALSE: Hide predictions. Used for pure manual annotation vs ML-assisted |
| **evaluate_predictions_automatically** | boolean | Auto-retrieve predictions | DEFAULT FALSE, DEPRECATED | **Legacy ML feature** - Automatically fetch predictions when loading task. Replaced by newer prediction system. Kept for backward compatibility |
| **token** | varchar(256) | Project invite token | DEFAULT hash, NULL | **Invite link security** - UUID/hash for project invite URLs. Independent of org/workspace tokens. Regeneratable via reset_token(). Used for adding members without workspace access |
| **result_count** | integer | Total result count | DEFAULT 0 | **Annotation counter** - Total annotation results across all task_completions. Updated via signals. Used for project progress tracking |
| **color** | varchar(16) | Project color code | DEFAULT '#FFFFFF', NULL | **UI customization** - Hex color for project badge/indicator. Helps visually distinguish projects in lists |
| **created_by_id** | integer | Creator user | FK to htx_user, NULL | **Ownership** - Who created project. NULL allowed (projects can exist without creator). Used for permission checks |
| **maximum_annotations** | integer | Max annotations per task | DEFAULT 1 | **CRITICAL: Overlap control** - How many annotations needed per task before marked complete:<br>- 1: Single annotation<br>- 2+: Multiple annotators (for agreement)<br>Triggers task.is_labeled=True when reached |
| **blind_passes** | integer | Blind pass count | DEFAULT 0 | **Quality control** - Number of matching annotations needed for automatic acceptance without review. Used in blind annotation workflow |
| **review_percentage** | integer | Review percentage | DEFAULT 0 | **QA sampling** - 0-100: Percentage of tasks sent to reviewers.<br>- 0: No review<br>- 50: Half of tasks reviewed<br>- 100: All tasks reviewed |
| **golden_ref_percentage** | integer | Golden reference percentage | DEFAULT 0 | **Quality benchmark** - 0-100: Percentage of tasks used as ground truth for quality metrics. Annotator performance measured against golden tasks |
| **min_annotations_to_start_training** | integer | Min annotations for ML | DEFAULT 0 | **ML trigger** - Minimum completed annotations before triggering ML backend training. 0 = disabled. Example: 100 means train model after 100 annotations |
| **control_weights** | jsonb | Annotation weights config | DEFAULT dict, NULL | **Agreement calculation** - JSON mapping control tags to weights for inter-annotator agreement:<br>Structure: `{control_name: {overall: 0.5, type: "Choices", labels: {"positive": 1.0}}}`<br>Auto-updated when label_config changes |
| **model_version** | text | Current model version | DEFAULT '', NULL | **ML model tracking** - Identifies which ML model version to use for predictions:<br>- Can be prediction.model_version<br>- Or ML backend title<br>Used to filter/display correct predictions |
| **data_types** | jsonb | Data types in project | DEFAULT dict, NULL | **Schema tracking** - JSON mapping data keys to types extracted from label_config:<br>Example: `{"image": "Image", "text": "Text"}`<br>Auto-updated when label_config changes |
| **is_draft** | boolean | Draft status | DEFAULT FALSE | **Project lifecycle** - TRUE: Project being set up, not ready. FALSE: Ready for use. Used to hide incomplete projects |
| **is_published** | boolean | Published status | DEFAULT FALSE | **Visibility control** - TRUE: Visible to annotators. FALSE: Only admins see it. Controls who can access project |
| **is_deleted** | boolean | Soft delete flag | DEFAULT FALSE | **Soft deletion** - TRUE: Project marked deleted but data preserved. FALSE: Active. Used for archival without data loss |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD, INDEXED | **Project creation time** - When project created. Indexed for sorting project lists by age |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification time. Auto-updates on any change |
| **sampling** | varchar(100) | Task sampling method | CHOICES, DEFAULT 'Sequential sampling' | **CRITICAL: Task distribution** - How tasks assigned to annotators:<br>- **SEQUENCE:** Order by Data Manager sorting<br>- **UNIFORM:** Random selection<br>- **UNCERTAINTY:** ML uncertainty scores (active learning)<br>- **EQUAL_DISTRIBUTION:** Balance tasks across annotators |
| **skip_queue** | varchar(100) | Skip queue behavior | CHOICES, DEFAULT 'REQUEUE_FOR_OTHERS' | **CRITICAL: Skip handling** - What happens when annotator skips task:<br>- **REQUEUE_FOR_ME:** Goes back to my queue<br>- **REQUEUE_FOR_OTHERS:** Goes to other annotators<br>- **IGNORE_SKIPPED:** Skip counts as completion<br>Affects task.is_labeled calculation |
| **show_ground_truth_first** | boolean | Prioritize ground truth | DEFAULT FALSE | **Task ordering** - TRUE: Show ground truth tasks before regular tasks. Used for training/calibration |
| **show_overlap_first** | boolean | Prioritize overlap tasks | DEFAULT FALSE | **Task ordering** - TRUE: Show tasks needing multiple annotations first. Ensures overlap tasks get annotated quickly |
| **overlap_cohort_percentage** | integer | Overlap cohort percentage | DEFAULT 100 | **Overlap distribution** - 0-100: What percentage of tasks get overlap:<br>- 100: All tasks get maximum_annotations<br>- 20: Only 20% get overlap, rest get 1 annotation<br>Used in _rearrange_overlap_cohort() |
| **task_data_login** | varchar(256) | Task data credentials | NULL | **External data access** - Username for protected external data sources (URLs in task data). Passed to fetch requests |
| **task_data_password** | varchar(256) | Task data password | NULL | **External data access** - Password paired with task_data_login. Used when task.data contains protected URLs |
| **pinned_at** | timestamp | Pin timestamp | NULL, INDEXED | **UI organization** - Non-NULL: Project pinned to top of lists. NULL: Normal sorting. Indexed for fast "show pinned first" queries |
| **state** | varchar(20) | Project state | CHOICES, DEFAULT 'pending' | **Project lifecycle** - Current state:<br>- **PENDING:** Created, not started<br>- **PAUSED:** Temporarily stopped<br>- **LIVE:** Active annotation<br>- **BUILDING_WORKFLOW:** Setup in progress<br>- **COMPLETED:** All tasks done |
| **responsible_person_id** | integer | Responsible user | FK to htx_user, SET_NULL, NULL | **Accountability** - User responsible for managing project in current state. Changes as project progresses. Used for routing notifications |

### Computed Fields (Added by Manager)

These fields added via `ProjectManager.with_counts()` annotations:
- **task_number**: Total tasks
- **finished_task_number**: Completed tasks (is_labeled=True)
- **total_predictions_number**: All predictions
- **total_annotations_number**: All annotations
- **num_tasks_with_annotations**: Tasks with ≥1 annotation
- **useful_annotation_number**: Non-skipped, non-ground-truth annotations
- **ground_truth_number**: Ground truth annotation count
- **skipped_annotations_number**: Cancelled/skipped annotations

### Business Logic Use Cases

**Primary Use Cases:**
- **Annotation Interface Definition:** XML label_config creates custom UI (bounding boxes, classifications, NER, etc.)
- **Quality Control:** Set maximum_annotations for overlap, review_percentage for QA sampling
- **Task Distribution:** Configure sampling method (random, sequential, active learning)
- **ML Integration:** Link ML backends via model_version, set training triggers
- **Project Lifecycle:** Move through states (pending → live → completed)
- **Multi-stage Workflows:** Use nesting for review/adjudication stages

**Critical Methods:**
- **validate_config(config_string, strict=False):** Validates label config against existing data
- **update_tasks_states(...):** Recalculates task.is_labeled after settings changes
- **get_parsed_config(autosave_cache=True):** Returns parsed label config with caching
- **get_model_versions(with_counters=False):** Lists all ML model versions used
- **_rearrange_overlap_cohort():** Redistributes overlap based on overlap_cohort_percentage
- **labeled_tasks():** Returns queryset of completed tasks
- **eta():** Calculates estimated completion time

**Configuration Example:**
```python
project = Project.objects.create(
    title="Sentiment Analysis - Customer Reviews",
    workspace=workspace,
    label_config="""
    <View>
        <Text name="review" value="$review_text"/>
        <Choices name="sentiment" toName="review" choice="single">
            <Choice value="positive"/>
            <Choice value="neutral"/>
            <Choice value="negative"/>
        </Choices>
        <Rating name="confidence" toName="review" maxRating="5"/>
    </View>
    """,
    expert_instruction="<h3>Guidelines</h3><ul><li>Consider overall tone</li><li>Ignore sarcasm detection</li></ul>",
    maximum_annotations=2,  # 2 annotators per task
    review_percentage=20,   # 20% go to reviewers
    sampling="UNIFORM",     # Random task assignment
    skip_queue="REQUEUE_FOR_OTHERS",  # Skips go to others
)
```

---

## 2. ProjectSummary

**Purpose:** Cached summary statistics for projects to avoid expensive aggregation queries
**Table Name:** `project_summary`
**Business Context:** Performance optimization. Pre-calculates expensive aggregations updated via signals

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **project_id** | integer | Project reference | PRIMARY KEY, FK to project, CASCADE | **One-to-one relationship** - Each project has exactly one summary. CASCADE delete removes summary when project deleted |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Cache creation time** - When summary record created (usually with project) |
| **all_data_columns** | jsonb | All data columns found | DEFAULT dict, NULL | **Schema discovery** - Maps column names to task counts:<br>Example: `{"image_url": 1000, "caption": 950, "metadata": 100}`<br>Shows which fields present in how many tasks. Updated on task import |
| **common_data_columns** | jsonb | Common data columns | DEFAULT list, NULL | **Schema intersection** - List of fields present in ALL tasks:<br>Example: `["image_url", "id"]`<br>Used to identify required fields. Recalculated when tasks imported |
| **created_annotations** | jsonb | Annotation type counts | DEFAULT dict, NULL | **Annotation statistics** - Maps annotation types to counts:<br>Example: `{("sentiment", "review", "Choices"): 500, ("bbox", "image", "RectangleLabels"): 1200}`<br>Structure: `{(from_name, to_name, type): count}`<br>Updated via increase/decrease_created_annotations |
| **created_labels** | jsonb | Label counts | DEFAULT dict, NULL | **Label statistics** - Maps control names to label counts:<br>Example: `{"sentiment": {"positive": 300, "negative": 150, "neutral": 50}}`<br>Shows label distribution. Updated on annotation save/delete |
| **created_labels_drafts** | jsonb | Draft label counts | DEFAULT dict, NULL | **Draft statistics** - Same structure as created_labels but for draft annotations. Tracks work-in-progress |

### Business Logic Use Cases

**Primary Use Cases:**
- **Project Overview Dashboard:** Show annotation progress without slow aggregations
- **Schema Validation:** Check if imported tasks have required fields
- **Label Distribution:** Show which labels most/least used
- **Progress Tracking:** Quick stats for project completion percentage

**Update Triggers:**
- **On Task Import:** update_data_columns() adds new column names
- **On Annotation Save:** increase_project_summary_counters() increments counts
- **On Annotation Delete:** decrease_project_summary_counters() decrements counts
- **On Draft Save:** update_created_labels_drafts() tracks draft labels

**Key Methods:**
- **reset(tasks_data_based=True):** Clears all or annotation-based stats
- **update_data_columns(tasks):** Updates column statistics from task list
- **remove_data_columns(tasks):** Decrements column statistics
- **update_created_annotations_and_labels(annotations):** Batch update annotation stats
- **remove_created_annotations_and_labels(annotations):** Batch remove annotation stats

**Performance Benefit:**
```python
# ❌ SLOW - aggregates on every request
positive_count = Annotation.objects.filter(
    project=project,
    result__contains=[{"value": {"choices": ["positive"]}}]
).count()

# ✅ FAST - read from cache
positive_count = project.summary.created_labels.get("sentiment", {}).get("positive", 0)
```

---

## 3. ProjectNestedRelations

**Purpose:** Parent-child project relationships for multi-stage annotation workflows
**Table Name:** `project_nested_relations`
**Business Context:** Enables review, adjudication, and quality control stages as separate projects

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Relation ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for parent-child relationship |
| **child_project_id** | integer | Child project | FK to project, CASCADE, NULL | **Child side** - Project receiving tasks from parent. CASCADE delete removes relation when child deleted |
| **parent_project_id** | integer | Parent project | FK to project, CASCADE, NULL | **Parent side** - Project sending tasks to child. CASCADE delete removes relation when parent deleted |
| **created_by_id** | integer | Creator user | FK to htx_user, NULL | **Relation creator** - Who set up nesting. Used for accountability |
| **nesting_rules** | jsonb | Nesting configuration | DEFAULT dict | **Workflow rules** - JSON config defining how tasks flow from parent to child:<br>- Which tasks move (completed, reviewed, etc.)<br>- When to move (on completion, on review, etc.)<br>- What data to copy<br>Example: `{"trigger": "on_review", "filter": {"review_status": "rejected"}}` |
| **is_active** | boolean | Active status | DEFAULT FALSE | **Relation toggle** - TRUE: Relation active, tasks flow. FALSE: Relation disabled, no task movement |

**Unique Constraint:** (child_project_id, parent_project_id) - One relation per parent-child pair

### Business Logic Use Cases

**Primary Use Cases:**
- **Review Stage:** Parent project → Child review project (reviewers see parent's completed tasks)
- **Adjudication:** Two parallel annotation projects → Adjudication project (resolve disagreements)
- **Multi-stage Pipeline:** Raw annotation → QA → Expert review (cascading stages)
- **Quality Control:** Main project → Gold standard verification project (check random sample)

**Workflow Examples:**

**Review Workflow:**
```python
# Setup
parent_project = Project.objects.create(title="Image Classification")
child_project = Project.objects.create(title="Review Stage")

ProjectNestedRelations.objects.create(
    parent_project=parent_project,
    child_project=child_project,
    nesting_rules={
        "trigger": "on_completion",
        "sample_percentage": 20,  # Send 20% to review
        "copy_annotations": False  # Reviewers annotate from scratch
    },
    is_active=True
)

# Flow:
# 1. Annotator completes task in parent_project
# 2. System checks if task selected for review (20% chance)
# 3. If yes, task appears in child_project queue
# 4. Reviewer annotates in child_project
```

**Adjudication Workflow:**
```python
# Setup: Two annotators disagree
project_a = Project.objects.create(title="Annotator A")
project_b = Project.objects.create(title="Annotator B")
adjudication_project = Project.objects.create(title="Adjudication")

# Both feed into adjudication
for parent in [project_a, project_b]:
    ProjectNestedRelations.objects.create(
        parent_project=parent,
        child_project=adjudication_project,
        nesting_rules={
            "trigger": "on_disagreement",
            "min_annotations": 2,
            "max_agreement": 0.7  # If agreement <70%, send to adjudication
        },
        is_active=True
    )
```

---

## SECTION 9: PROJECT CONFIGURATION & MANAGEMENT

---

## 4. projects_tag

**Purpose:** Reusable tags for categorizing and filtering projects
**Table Name:** `projects_tag`
**Business Context:** Taxonomy system for organizing projects by type, language, domain, etc.

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Tag ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for tag |
| **name** | varchar(255) | Tag name | NOT NULL | **Tag label** - Display name (e.g., "Spanish", "Medical", "Q1-2024", "High-Priority"). Used in filters and project badges |
| **tag_type** | varchar(50) | Tag category | NULL | **Tag taxonomy** - Groups tags by type:<br>- "language": Spanish, French, English<br>- "domain": Medical, Legal, Financial<br>- "priority": High, Medium, Low<br>- "quarter": Q1-2024, Q2-2024<br>Enables hierarchical filtering |
| **color** | varchar(7) | Color code | DEFAULT '#FFFFFF' | **Visual indicator** - Hex color for tag badge. Helps visually distinguish tag types (languages=blue, priority=red) |

### Business Logic Use Cases

**Primary Use Cases:**
- **Project Filtering:** "Show all Spanish Medical projects"
- **Project Categorization:** Organize 100+ projects by domain/language
- **Reporting:** Group projects by tag for analytics
- **Access Control:** Future use for tag-based permissions

**Tag Organization Example:**
```
Language Tags (blue):
- Spanish (#3B82F6)
- French (#3B82F6)
- Mandarin (#3B82F6)

Domain Tags (green):
- Medical (#10B981)
- Legal (#10B981)
- Finance (#10B981)

Priority Tags (red):
- Critical (#EF4444)
- High (#F97316)
- Normal (#6B7280)

Quarter Tags (purple):
- Q1-2024 (#8B5CF6)
- Q2-2024 (#8B5CF6)
```

---

## 5. ProjectTag

**Purpose:** Links projects to tags (many-to-many relationship)
**Table Name:** `project_tag`
**Business Context:** Through table enabling projects to have multiple tags

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Link ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for project-tag association |
| **project_id** | integer | Project reference | FK to project, CASCADE | **Project side** - Which project is tagged. CASCADE delete removes link when project deleted |
| **tag_id** | integer | Tag reference | FK to projects_tag, CASCADE | **Tag side** - Which tag applies. CASCADE delete removes link when tag deleted |

**Unique Constraint:** (project_id, tag_id) - One tag per project (no duplicates)

### Business Logic Use Cases

**Primary Use Cases:**
- **Multi-tag Projects:** Project tagged with both "Spanish" and "Medical" and "Q1-2024"
- **Bulk Tagging:** Tag 50 projects with "High-Priority" in one operation
- **Tag-based Filters:** Show all projects with tag="Medical" AND tag="High-Priority"

**Usage Example:**
```python
# Create project
project = Project.objects.create(title="Medical Spanish Translation")

# Add multiple tags
spanish_tag = Tag.objects.get(name="Spanish")
medical_tag = Tag.objects.get(name="Medical")
q1_tag = Tag.objects.get(name="Q1-2024")

for tag in [spanish_tag, medical_tag, q1_tag]:
    ProjectTag.objects.create(project=project, tag=tag)

# Query: Find all Medical Spanish projects
Project.objects.filter(
    project_tags__tag__name="Medical"
).filter(
    project_tags__tag__name="Spanish"
)
```

---

## 6. ProjectClientApiConfiguration

**Purpose:** External API configurations for client-side data generation (TTS, translation, etc.)
**Table Name:** `project_client_api_configuration`
**Business Context:** Enables calling external APIs from browser to generate/augment task data

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Config ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for API configuration |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project scope** - Which project uses this API. CASCADE delete removes config when project deleted |
| **data_key** | varchar(50) | Field name in task data | NOT NULL | **Target field** - Which field in task.data to populate:<br>Example: "audio_url", "translation", "summary"<br>Used to map API response to task data structure |
| **group** | varchar(100) | Configuration group | DEFAULT "group1" | **Multi-config support** - Groups multiple API configs:<br>- "group1": Primary TTS provider<br>- "group2": Fallback TTS provider<br>Enables A/B testing, failover |
| **client_url** | varchar(1000) | API endpoint | NOT NULL | **API endpoint** - Full URL to external service:<br>- TTS: "https://api.elevenlabs.io/v1/text-to-speech"<br>- Translation: "https://api.deepl.com/v2/translate"<br>- OCR: "https://api.mathpix.com/v3/text" |
| **headers** | jsonb | API headers | NULL | **Authentication & config** - HTTP headers for API request:<br>Example: `{"Authorization": "Bearer sk-...", "Content-Type": "application/json"}`<br>Supports API keys, auth tokens |
| **payload** | jsonb | API request template | NULL | **Request body** - JSON template for API request:<br>Example: `{"text": "$text", "voice_id": "adam", "model_id": "eleven_multilingual_v2"}`<br>Variables like $text replaced with task data |

**Unique Constraint:** (project, group, data_key) - One config per project-group-field combination

### Business Logic Use Cases

**Primary Use Cases:**
- **Text-to-Speech:** Generate audio files from text for audio annotation projects
- **Machine Translation:** Pre-translate text for translation review projects
- **OCR:** Extract text from images for text annotation projects
- **Synthetic Data:** Generate synthetic examples for training data

**TTS Configuration Example:**
```python
ProjectClientApiConfiguration.objects.create(
    project=project,
    data_key="audio_url",
    group="elevenlabs_primary",
    client_url="https://api.elevenlabs.io/v1/text-to-speech/21m00Tcm4TlvDq8ikWAM",
    headers={
        "xi-api-key": "your_api_key_here",
        "Content-Type": "application/json"
    },
    payload={
        "text": "$text",  # Replaced with task.data['text']
        "model_id": "eleven_multilingual_v2",
        "voice_settings": {
            "stability": 0.5,
            "similarity_boost": 0.75
        }
    }
)

# Usage: When FileUpload reads task with text="Hello world"
# System calls API, gets audio file, sets task.data['audio_url'] = returned_url
```

**Translation Configuration:**
```python
ProjectClientApiConfiguration.objects.create(
    project=project,
    data_key="translation",
    group="deepl",
    client_url="https://api.deepl.com/v2/translate",
    headers={
        "Authorization": "DeepL-Auth-Key your_key",
        "Content-Type": "application/json"
    },
    payload={
        "text": ["$source_text"],
        "target_lang": "ES",
        "source_lang": "EN"
    }
)
```

---

## 7. ProjectOnboardingSteps

**Purpose:** Defines onboarding checklist steps shown to project creators
**Table Name:** `project_onboarding_steps`
**Business Context:** Guides users through project setup with progressive disclosure

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Step ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for onboarding step |
| **code** | varchar(2) | Step code | NOT NULL | **Step identifier** - Short code for step (e.g., "PR", "IM", "AN", "RV"). Used for programmatic checks |
| **title** | varchar(1000) | Step title | NOT NULL | **Display title** - User-facing name (e.g., "Configure Labeling Interface", "Import Tasks", "Add Team Members") |
| **description** | text | Step description | NULL | **Step instructions** - Detailed explanation of what to do in this step. Can include HTML for formatting |
| **order** | integer | Display order | NOT NULL | **Sort order** - Determines sequence of steps (1, 2, 3...). Steps shown in ascending order |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Step creation time** |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - When step definition last modified |

### Business Logic Use Cases

**Primary Use Cases:**
- **Project Setup Wizard:** Step-by-step guide for new project creators
- **Progress Tracking:** Show "3 of 7 steps completed"
- **Customizable Onboarding:** Admins can add/edit/reorder steps

**Standard Onboarding Flow:**
```
1. PR - Project Settings (title, description, instructions)
2. LC - Labeling Configuration (XML interface setup)
3. IM - Import Tasks (upload data)
4. TM - Add Team Members (invite annotators)
5. QL - Quality Settings (overlap, review percentage)
6. ML - ML Integration (optional - connect ML backend)
7. LN - Launch Project (publish to annotators)
```

---

## 8. ProjectOnboarding

**Purpose:** Tracks which onboarding steps completed per project
**Table Name:** `project_onboarding`
**Business Context:** Per-project completion status for onboarding checklist

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Record ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for completion record |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project reference** - Which project this completion is for. CASCADE delete removes records when project deleted |
| **step_id** | integer | Onboarding step | FK to ProjectOnboardingSteps, CASCADE | **Step reference** - Which step is completed. CASCADE delete removes records when step deleted |
| **finished** | boolean | Completion status | DEFAULT FALSE | **Completion flag** - TRUE: Step completed. FALSE: Not done yet. Used to show checkmarks in UI |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Record creation** - When tracking started for this step |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Completion time** - When finished was set to TRUE |

### Business Logic Use Cases

**Primary Use Cases:**
- **Progress Display:** Show "Step 3 of 7 completed" with checkmarks
- **Auto-detection:** Some steps auto-complete (e.g., "Import Tasks" when first task imported)
- **Guided Setup:** Hide advanced features until basic steps completed

**Auto-completion Logic:**
```python
# When tasks imported
if project.task_set.count() > 0:
    ProjectOnboarding.objects.update_or_create(
        project=project,
        step=ProjectOnboardingSteps.objects.get(code="IM"),
        defaults={"finished": True}
    )

# When team members added
if ProjectMember.objects.filter(project=project).count() > 1:
    ProjectOnboarding.objects.update_or_create(
        project=project,
        step=ProjectOnboardingSteps.objects.get(code="TM"),
        defaults={"finished": True}
    )
```

---

## 9. ProjectMember

**Purpose:** Links users to projects with specific roles and task limits
**Table Name:** `project_member`
**Business Context:** Controls who can access project and their permissions

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Membership ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for project membership |
| **user_id** | integer | Member user | FK to htx_user, CASCADE | **User reference** - Which user is project member. CASCADE delete removes membership when user deleted |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project reference** - Which project user can access. CASCADE delete removes membership when project deleted |
| **enabled** | boolean | Membership enabled | DEFAULT TRUE | **Soft disable** - TRUE: Member active. FALSE: Temporarily disabled without removal. Can re-enable without recreating membership |
| **member_type** | integer | Member type/role | DEFAULT 1 (ANNOTATOR) | **CRITICAL: Role-based permissions:**<br>- **1 (ADMIN):** Full project control, edit settings, manage members<br>- **2 (ADJUDICATOR):** Resolve conflicts between annotators<br>- **3 (REVIEWER):** Review and approve/reject annotations<br>- **4 (ANNOTATOR):** Create annotations on tasks |
| **task_limit** | integer (PositiveIntegerField) | Task limit | DEFAULT 0 | **Workload cap** - Maximum tasks this member can annotate:<br>- 0: Unlimited<br>- 100: Can annotate up to 100 tasks<br>Used to distribute work fairly or limit contractors |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Membership start** - When user added to project |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - When role or settings changed |

**Unique Constraint:** (user, project) - One membership per user per project

### Business Logic Use Cases

**Primary Use Cases:**
- **Team Assignment:** Add annotators/reviewers to project
- **Role Management:** Promote annotator to reviewer
- **Workload Distribution:** Set task_limit to distribute fairly among team
- **Temporary Disable:** Set enabled=FALSE for vacation/leave

**Task Limit Workflow:**
```python
# Setup: Project with 1000 tasks, 10 annotators
project = Project.objects.get(id=1)

# Distribute evenly: each annotator gets 100 tasks max
for user in annotators:
    ProjectMember.objects.create(
        project=project,
        user=user,
        member_type=ANNOTATOR,
        task_limit=100  # Fair distribution
    )

# Task assignment respects limits:
def get_next_task(user, project):
    user_annotation_count = Annotation.objects.filter(
        completed_by=user,
        project=project
    ).count()

    member = ProjectMember.objects.get(user=user, project=project)
    if member.task_limit > 0 and user_annotation_count >= member.task_limit:
        return None  # User reached limit

    return Task.objects.filter(project=project, ...).first()
```

---

## 10. ProjectGroup

**Purpose:** Folder/category organization for grouping projects in workspace
**Table Name:** `project_group`
**Business Context:** Virtual folders for organizing projects in UI (like filesystem folders)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Group ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for project group |
| **name** | varchar(255) | Group name | NOT NULL | **Folder name** - Display name (e.g., "Q1 Projects", "Medical Team", "Client X"). Shown in project tree UI |
| **parent_id** | integer | Parent group | FK to self, CASCADE, NULL | **Nested folders** - Self-referencing FK for folder hierarchy:<br>- NULL: Root-level folder<br>- Non-NULL: Subfolder<br>Example: "2024" → "Q1" → "Medical" |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE | **Workspace scope** - Groups belong to workspace. CASCADE delete removes groups when workspace deleted |
| **projects** | ManyToManyField | Projects in group | Through ProjectGroupProject | **Group membership** - Which projects in this folder. Many-to-many allows project in multiple groups |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD, NULL (backward compat) | **Group creation time** - Nullable for backward compatibility with old records |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW, NULL (backward compat) | **Change tracking** - Nullable for backward compatibility |

**Unique Constraint:** (workspace, name, parent) - Folder names unique within parent

### Business Logic Use Cases

**Primary Use Cases:**
- **Project Organization:** Group 100+ projects into folders
- **Nested Hierarchy:** Create folder trees (Year → Quarter → Department)
- **Multi-category:** Project can be in multiple groups (cross-link)
- **Permission Inheritance:** (Future) Permissions on folder apply to projects inside

**Folder Structure Example:**
```
Workspace: Production
├─ 2024/
│  ├─ Q1/
│  │  ├─ Medical Projects (3 projects)
│  │  └─ Legal Projects (5 projects)
│  └─ Q2/
│     └─ New Initiatives (2 projects)
├─ Clients/
│  ├─ Client A (10 projects)
│  └─ Client B (8 projects)
└─ Archives/ (old projects)
```

**Query Example:**
```python
# Get all projects in "Q1 Medical" including subfolders
q1_medical = ProjectGroup.objects.get(name="Q1/Medical")
projects = Project.objects.filter(
    project_groups__in=[q1_medical] + list(q1_medical.get_descendants())
)
```

---

## 11. ProjectOpsManager

**Purpose:** Task distribution strategy configuration per project
**Table Name:** `project_ops_manager`
**Business Context:** Controls how tasks distributed among annotators (equal, max limit, custom rules)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Config ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for ops config |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project scope** - Which project uses this distribution strategy. CASCADE delete removes config when project deleted |
| **max_limit** | integer (PositiveIntegerField) | Maximum task limit | NOT NULL | **Default max** - Default maximum tasks per annotator when using MAX_LIMIT strategy |
| **split_strategy** | varchar(20) | Distribution strategy | CHOICES, NOT NULL | **CRITICAL: Task distribution algorithm:**<br>- **EQUAL:** Distribute tasks evenly among all annotators<br>- **MAX_LIMIT:** Each annotator gets up to max_limit tasks<br>- **USER_LIMITS:** Custom limit per user (from user_book)<br>- **ORGANIZATION_LIMITS:** Custom limit per organization (from rule_book) |
| **rule_book** | jsonb | Organization-specific limits | DEFAULT dict | **Org-level rules** - JSON mapping organization IDs to task limits:<br>Example: `{5: 1000, 12: 500}`<br>Org 5 gets max 1000 tasks, Org 12 gets 500<br>Used with ORGANIZATION_LIMITS strategy |
| **user_book** | jsonb | User-specific limits | DEFAULT dict | **User-level rules** - JSON mapping user IDs to task limits:<br>Example: `{42: 200, 108: 100, 203: 50}`<br>User 42 gets 200 tasks, User 108 gets 100, etc.<br>Used with USER_LIMITS strategy |

### Business Logic Use Cases

**Primary Use Cases:**
- **Equal Distribution:** Ensure all annotators get same workload
- **Capacity-based:** Assign based on annotator availability/skill
- **Organization Quotas:** Limit tasks per client organization
- **Custom Allocation:** Fine-grained control per user

**get_limit(user) Method:**
```python
def get_limit(self, user):
    """Returns task limit for user based on strategy"""
    if self.split_strategy == "EQUAL":
        total_tasks = self.project.tasks.count()
        total_annotators = ProjectMember.objects.filter(
            project=self.project,
            member_type=ANNOTATOR
        ).count()
        return total_tasks // total_annotators

    elif self.split_strategy == "MAX_LIMIT":
        return self.max_limit

    elif self.split_strategy == "USER_LIMITS":
        return self.user_book.get(str(user.id), self.max_limit)

    elif self.split_strategy == "ORGANIZATION_LIMITS":
        org_id = user.active_organization_id
        return self.rule_book.get(str(org_id), self.max_limit)
```

**Configuration Examples:**

**Equal Distribution:**
```python
ProjectOpsManager.objects.create(
    project=project,
    split_strategy="EQUAL",
    max_limit=0  # Not used in EQUAL mode
)
# Result: 1000 tasks ÷ 10 annotators = 100 tasks each
```

**User-specific Limits:**
```python
ProjectOpsManager.objects.create(
    project=project,
    split_strategy="USER_LIMITS",
    max_limit=100,  # Default for users not in user_book
    user_book={
        "42": 200,   # Senior annotator: 200 tasks
        "108": 150,  # Mid-level: 150 tasks
        "203": 50    # Junior: 50 tasks
    }
)
```

**Organization Quotas:**
```python
ProjectOpsManager.objects.create(
    project=project,
    split_strategy="ORGANIZATION_LIMITS",
    max_limit=100,
    rule_book={
        "5": 5000,   # Premium client org: 5000 tasks
        "12": 1000,  # Standard client: 1000 tasks
        "18": 500    # Small client: 500 tasks
    }
)
```

---

## 12. ProjectEmbedding

**Purpose:** Embedded external dashboards in project UI
**Table Name:** `project_embedding`
**Business Context:** Enables embedding analytics dashboards, reports, or tools within project interface

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Embedding ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for embedding configuration |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project scope** - Which project has this embedding. CASCADE delete removes embedding when project deleted |
| **title** | varchar(50) | Dashboard title | NULL | **Display name** - Tab/button label (e.g., "Quality Metrics", "Annotation Speed", "Label Distribution") |
| **iframe_url** | varchar(1000) | Embed URL | NULL | **Dashboard source** - Full URL to external dashboard:<br>- Grafana: "https://grafana.company.com/d-solo/xyz"<br>- Looker: "https://looker.company.com/embed/looks/42"<br>- Custom: "https://analytics.company.com/project/123" |
| **button_text** | varchar(50) | Button label | NULL | **CTA text** - Button text to open dashboard (e.g., "View Metrics", "Open Dashboard", "Analytics") |
| **visible** | boolean | Visibility flag | DEFAULT FALSE | **Toggle** - TRUE: Show in UI. FALSE: Hidden. Allows setup/testing without showing to users |
| **order** | integer (PositiveIntegerField) | Display order | DEFAULT 0 | **Sort order** - For multiple dashboards, determines tab/menu order (0, 1, 2...) |
| **width** | integer (PositiveIntegerField) | Iframe width | DEFAULT 1200 | **Dimensions** - Width in pixels for embedded iframe |
| **height** | integer (PositiveIntegerField) | Iframe height | DEFAULT 600 | **Dimensions** - Height in pixels for embedded iframe |

### Business Logic Use Cases

**Primary Use Cases:**
- **Quality Dashboards:** Embed Grafana showing inter-annotator agreement
- **Progress Tracking:** Looker dashboard showing completion over time
- **Custom Reports:** Project-specific analytics from external BI tools
- **Integration Tools:** Embed project management tools (Jira, Asana)

**Example Configurations:**

**Grafana Quality Metrics:**
```python
ProjectEmbedding.objects.create(
    project=project,
    title="Quality Metrics",
    iframe_url="https://grafana.company.com/d-solo/abc123/quality?orgId=1&panelId=2&from=now-7d&to=now&var-project=123",
    button_text="View Quality Dashboard",
    visible=True,
    order=1,
    width=1400,
    height=800
)
```

**Looker Progress Dashboard:**
```python
ProjectEmbedding.objects.create(
    project=project,
    title="Progress Report",
    iframe_url="https://looker.company.com/embed/dashboards/42?Project+ID=123",
    button_text="Progress Dashboard",
    visible=True,
    order=2,
    width=1200,
    height=600
)
```

---

## 13. TemplateDashboard

**Purpose:** Reusable dashboard templates for projects
**Table Name:** `template_dashboard`
**Business Context:** Pre-configured dashboard templates that can be applied to multiple projects

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Template ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for dashboard template |
| **template_type** | varchar(200) | Template type identifier | NOT NULL | **Template category** - Identifies template type (e.g., "quality_metrics", "progress_tracking", "label_distribution"). Used to filter templates by use case |
| **code** | text | Template code/config | NULL | **Template definition** - Code or configuration for dashboard:<br>- Could be dashboard-as-code (JSON/YAML)<br>- SQL queries for reports<br>- Chart configurations |
| **dashboard_link** | varchar(200) | Template dashboard URL | NULL | **Template URL** - URL pattern for dashboard with placeholders:<br>Example: "https://grafana.com/d/quality?project={{project_id}}"<br>Placeholders replaced when applied to project |
| **title** | varchar(50) | Template title | NULL | **Display name** - Template name shown in template picker |
| **button_text** | varchar(50) | Button label | NULL | **Default CTA** - Default button text when applied to project |
| **visible** | boolean | Visibility flag | DEFAULT FALSE | **Template status** - TRUE: Available for selection. FALSE: Hidden/deprecated |
| **order** | integer (PositiveIntegerField) | Display order | DEFAULT 0 | **Sort order** - Order in template picker list |
| **width** | integer (PositiveIntegerField) | Default width | DEFAULT 1200 | **Default dimensions** - Width when applied to project |
| **height** | integer (PositiveIntegerField) | Default height | DEFAULT 600 | **Default dimensions** - Height when applied to project |

### Business Logic Use Cases

**Primary Use Cases:**
- **Standard Dashboards:** Apply "Quality Metrics" template to all new projects
- **Template Library:** Curated collection of useful dashboards
- **Quick Setup:** One-click add dashboard to project
- **Consistency:** Ensure all projects have same reporting standards

**Template Application Workflow:**
```python
# 1. Admin creates template
template = TemplateDashboard.objects.create(
    template_type="quality_metrics",
    title="Inter-Annotator Agreement",
    dashboard_link="https://grafana.com/d/quality?project={{project_id}}&org={{org_id}}",
    button_text="Quality Dashboard",
    visible=True
)

# 2. User applies template to project
def apply_template(project, template):
    # Replace placeholders
    url = template.dashboard_link.replace("{{project_id}}", str(project.id))
    url = url.replace("{{org_id}}", str(project.workspace.organization_id))

    # Create ProjectEmbedding from template
    ProjectEmbedding.objects.create(
        project=project,
        title=template.title,
        iframe_url=url,
        button_text=template.button_text,
        visible=True,
        width=template.width,
        height=template.height
    )
```

---

## SECTION 10: PROJECT DATA IMPORT/EXPORT

---

## 14. ProjectImport

**Purpose:** Tracks bulk task import jobs with progress and error handling
**Table Name:** `project_import`
**Business Context:** Asynchronous task import from files (CSV, JSON), URLs, or cloud storage

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Import job ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for import job |
| **project_id** | integer | Target project | FK to project, CASCADE | **Project reference** - Where tasks are imported. CASCADE delete removes import records when project deleted |
| **preannotated_from_fields** | jsonb | Pre-annotation field mapping | NULL | **Pre-annotation config** - Maps task data fields to annotations:<br>Example: `["prediction"]` means use task.data['prediction'] as pre-annotation<br>Converts predictions to annotations automatically |
| **commit_to_project** | boolean | Auto-commit flag | DEFAULT TRUE | **Staging control** - TRUE: Tasks immediately added to project. FALSE: Staged for review first (preview mode) |
| **return_task_ids** | boolean | Return task IDs flag | DEFAULT FALSE | **Response config** - TRUE: Return list of created task IDs. FALSE: Return summary only. TRUE increases response size but useful for tracking |
| **status** | varchar(64) | Import job status | NOT NULL | **CRITICAL: Job state** - Current status:<br>- "created": Job queued<br>- "in_progress": Currently importing<br>- "completed": All tasks imported<br>- "failed": Error occurred<br>Used to show progress in UI |
| **url** | varchar(2048) | Source URL | NULL | **External data source** - URL to fetch data from:<br>- CSV/JSON file URL<br>- Cloud storage URL (S3, GCS, Azure)<br>- API endpoint returning task list |
| **traceback** | text | Error traceback | NULL | **Error details** - Full Python traceback if import failed. Used for debugging |
| **error** | text | Error message | NULL | **User-friendly error** - Short error description shown to user |
| **created_at** | timestamp | Job creation time | AUTO_NOW_ADD | **Import start time** - When import job created |
| **updated_at** | timestamp | Last update time | AUTO_NOW | **Progress tracking** - Updates as import progresses. Used to detect stuck imports |
| **finished_at** | timestamp | Completion time | NULL | **Import end time** - When import completed/failed. Used to calculate duration |
| **task_count** | integer | Total tasks imported | DEFAULT 0 | **Success count** - How many tasks successfully created |
| **annotation_count** | integer | Total annotations imported | DEFAULT 0 | **Annotation count** - Pre-annotations created from preannotated_from_fields |
| **prediction_count** | integer | Total predictions imported | DEFAULT 0 | **Prediction count** - Predictions created from task data |
| **duration** | integer | Import duration | DEFAULT 0 | **Performance metric** - Seconds taken to complete import. Used for capacity planning |
| **file_upload_ids** | jsonb | Associated file uploads | NULL | **File tracking** - List of FileUpload IDs that triggered this import:<br>Example: `[42, 43, 44]` for batch import from 3 files |
| **could_be_tasks_list** | boolean | Tasks list flag | DEFAULT True | **Import mode detection** - TRUE: File contains list of tasks. FALSE: File itself becomes task (e.g., single image file) |
| **found_formats** | jsonb | Detected data formats | NULL | **Format detection** - Which data types found in import:<br>Example: `{"Image": 100, "Text": 100}` means 100 tasks with images and text |
| **data_columns** | jsonb | Detected column names | NULL | **Schema discovery** - Column names found in import file:<br>Example: `["image_url", "caption", "labels"]` |
| **tasks** | jsonb | Staged tasks data | NULL | **Preview data** - When commit_to_project=FALSE, staged tasks stored here for preview |
| **task_ids** | jsonb | Created task IDs | NULL | **Created task tracking** - List of task IDs created by this import. Only populated if return_task_ids=TRUE |

### Business Logic Use Cases

**Primary Use Cases:**
- **Bulk Task Import:** Upload CSV with 10,000 tasks
- **Pre-annotation Import:** Import tasks with ML predictions
- **Progress Tracking:** Show "Imported 5,000 of 10,000 tasks (50%)"
- **Error Recovery:** Retry failed imports from last successful point
- **Preview Mode:** Review tasks before committing to project

**Import Workflow:**
```python
# 1. User uploads CSV file
file_upload = FileUpload.objects.create(user=user, project=project, file=csv_file)

# 2. System creates import job
import_job = ProjectImport.objects.create(
    project=project,
    commit_to_project=True,
    return_task_ids=False,
    status="created",
    file_upload_ids=[file_upload.id]
)

# 3. Background worker processes import
def process_import(import_job):
    import_job.status = "in_progress"
    import_job.save()

    try:
        tasks_data = parse_csv(import_job.file_upload_ids[0])
        import_job.data_columns = list(tasks_data[0].keys())

        for i, task_data in enumerate(tasks_data):
            Task.objects.create(project=import_job.project, data=task_data)
            import_job.task_count = i + 1
            import_job.save()  # Update progress

        import_job.status = "completed"
        import_job.finished_at = now()
        import_job.duration = (import_job.finished_at - import_job.created_at).seconds
        import_job.save()

    except Exception as e:
        import_job.status = "failed"
        import_job.error = str(e)
        import_job.traceback = traceback.format_exc()
        import_job.save()
```

**Pre-annotation Import:**
```python
# CSV contains: image_url, prediction
# prediction column: {"label": "cat", "score": 0.95}

ProjectImport.objects.create(
    project=project,
    preannotated_from_fields=["prediction"],  # Use prediction column
    commit_to_project=True,
    file_upload_ids=[file_upload.id]
)

# Result: Tasks created with annotations from prediction column
```

---

## 15. ProjectReimport

**Purpose:** Re-imports tasks to update existing data without creating duplicates
**Table Name:** `project_reimport`
**Business Context:** Updates existing tasks when source data changes (e.g., re-run ML predictions)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Reimport job ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for reimport job |
| **project_id** | integer | Target project | FK to project, CASCADE | **Project reference** - Which project's tasks to update. CASCADE delete removes reimport records when project deleted |
| **status** | varchar(64) | Reimport job status | NOT NULL | **Job state** - Same as ProjectImport: created/in_progress/completed/failed |
| **error** | text | Error message | NULL | **User-friendly error** - Short error description |
| **task_count** | integer | Tasks updated | DEFAULT 0 | **Update count** - How many tasks updated (not created) |
| **annotation_count** | integer | Annotations updated | DEFAULT 0 | **Annotation updates** - Annotations modified during reimport |
| **prediction_count** | integer | Predictions updated | DEFAULT 0 | **Prediction updates** - Predictions created/updated |
| **duration** | integer | Reimport duration | DEFAULT 0 | **Performance metric** - Seconds taken |
| **file_upload_ids** | jsonb | Associated file uploads | NULL | **File tracking** - Source files for reimport |
| **files_as_tasks_list** | boolean | Import mode flag | DEFAULT True | **Import mode** - Same as ProjectImport.could_be_tasks_list |
| **found_formats** | jsonb | Detected formats | NULL | **Format detection** - Data types found |
| **data_columns** | jsonb | Column names | NULL | **Schema** - Columns in reimport file |
| **traceback** | text | Error traceback | NULL | **Debug info** - Full error traceback |

### Business Logic Use Cases

**Primary Use Cases:**
- **Update ML Predictions:** Re-run predictions on existing tasks
- **Correct Task Data:** Fix errors in task.data without recreating tasks
- **Add Missing Fields:** Add new columns to existing tasks
- **Refresh External Data:** Update data fetched from URLs

**Reimport vs Import:**
- **Import (ProjectImport):** Creates NEW tasks, skips duplicates
- **Reimport (ProjectReimport):** Updates EXISTING tasks, doesn't create new

**Reimport Workflow:**
```python
# Scenario: ML model improved, want to update predictions

# 1. Export task IDs and data
existing_tasks = Task.objects.filter(project=project).values('id', 'data')

# 2. Generate new predictions externally
updated_predictions = run_ml_model(existing_tasks)

# 3. Create reimport file with task IDs + new predictions
# CSV format: task_id, image_url, new_prediction
reimport_csv = create_csv(updated_predictions)

# 4. Upload and reimport
file_upload = FileUpload.objects.create(...)
ProjectReimport.objects.create(
    project=project,
    file_upload_ids=[file_upload.id],
    status="created"
)

# 5. System matches tasks by ID and updates predictions
# Tasks remain same, only predictions updated
```

---

## 16. FileUpload

**Purpose:** Tracks uploaded files for task import with processing statistics
**Table Name:** `io_fileupload`
**Business Context:** Manages file uploads and tracks import success/failure rates

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Upload ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for file upload |
| **user_id** | integer | Uploader | FK to htx_user, CASCADE | **User tracking** - Who uploaded file. CASCADE delete removes uploads when user deleted |
| **project_id** | integer | Target project | FK to project, CASCADE | **Project reference** - Where file imported. CASCADE delete removes uploads when project deleted |
| **file** | varchar(100) (FileField) | Uploaded file path | NOT NULL | **File location** - Path to uploaded file on storage (local or cloud). FileField handles upload/storage |
| **serial_number** | varchar(100) | Optional tracking ID | NULL | **External reference** - Optional user-provided identifier for tracking (e.g., batch number, order ID) |
| **total_rows** | integer (PositiveIntegerField) | Total rows in file | DEFAULT 0 | **Row count** - Total rows/records in uploaded file. Set after parsing |
| **successful_rows** | integer (PositiveIntegerField) | Successfully imported | DEFAULT 0 | **Success count** - How many rows imported without errors |
| **unsuccessful_rows** | integer (PositiveIntegerField) | Failed imports | DEFAULT 0 | **Failure count** - How many rows failed validation/import. total_rows = successful_rows + unsuccessful_rows |
| **uploaded_at** | timestamp | Upload timestamp | AUTO_NOW_ADD | **Upload time** - When file was uploaded |

### Business Logic Use Cases

**Primary Use Cases:**
- **Import Progress:** Show "Imported 950 of 1000 rows (95% success)"
- **Error Detection:** Identify files with high failure rates
- **Batch Tracking:** Use serial_number to track import batches
- **Audit Trail:** Track who uploaded what when

**File Processing Workflow:**
```python
# 1. User uploads CSV via UI
file_upload = FileUpload.objects.create(
    user=request.user,
    project=project,
    file=uploaded_file,
    serial_number="BATCH_2024_Q1_001"
)

# 2. Parse file to count rows
rows = parse_csv(file_upload.file)
file_upload.total_rows = len(rows)
file_upload.save()

# 3. Import with tracking
success_count = 0
failure_count = 0

for row in rows:
    try:
        Task.objects.create(project=project, data=row)
        success_count += 1
    except ValidationError:
        failure_count += 1

    # Update progress every 100 rows
    if (success_count + failure_count) % 100 == 0:
        file_upload.successful_rows = success_count
        file_upload.unsuccessful_rows = failure_count
        file_upload.save()

# 4. Final update
file_upload.successful_rows = success_count
file_upload.unsuccessful_rows = failure_count
file_upload.save()

# 5. Show results to user
print(f"Imported {success_count}/{file_upload.total_rows} rows")
print(f"Failed: {failure_count} rows")
```

**File Type Handlers:**
- **read_tasks_list_from_csv():** Parse CSV files
- **read_tasks_list_from_tsv():** Parse TSV files
- **read_tasks_list_from_json():** Parse JSON arrays
- **read_tasks_list_from_txt():** One task per line
- **read_task_from_hypertext_body():** HTML/XML as single task
- **read_task_from_uploaded_file():** File itself becomes task (e.g., image)
- **generate_and_save_audio():** TTS integration for audio generation

---

## SECTION 17: LABELS & TAGS

---

## 17. Label

**Purpose:** Reusable label definitions for consistent labeling across projects
**Table Name:** `label`
**Business Context:** Label library for standardized labels (e.g., company-wide sentiment labels)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Label ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for label |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Label creation time** |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - When label definition changed |
| **created_by_id** | integer | Creator user | FK to htx_user, SET_NULL | **Label author** - Who created label. SET_NULL preserves label if user deleted |
| **value** | jsonb | Label value/config | NOT NULL | **Label definition** - JSON structure defining label:<br>Example: `{"choices": ["positive", "negative", "neutral"]}`<br>Or: `{"rectanglelabels": ["person", "car", "dog"]}` |
| **title** | varchar(2048) | Label title | NOT NULL | **Display name** - Human-readable label name (e.g., "Sentiment Labels", "Object Detection Classes") |
| **description** | text | Label description | NULL | **Documentation** - Explains when to use this label, guidelines, examples |
| **approved_by_id** | integer | Approver user | FK to htx_user, SET_NULL | **QA approval** - Who approved label for use. NULL if not approved |
| **approved** | boolean | Approval status | DEFAULT FALSE | **Quality gate** - TRUE: Approved for use. FALSE: Draft/pending review. Can restrict unapproved labels |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE | **Workspace scope** - Labels belong to workspace. CASCADE delete removes labels when workspace deleted |

### Business Logic Use Cases

**Primary Use Cases:**
- **Label Standardization:** Ensure all sentiment projects use same labels
- **Label Library:** Curated collection of approved labels
- **Reusability:** Define once, use in multiple projects
- **Quality Control:** Approval workflow before labels used in production

**Label Library Workflow:**
```python
# 1. User creates label definition
sentiment_label = Label.objects.create(
    title="Standard Sentiment Labels",
    description="Company-wide sentiment classification labels",
    value={
        "choices": ["positive", "negative", "neutral"],
        "instructions": "positive = happy/satisfied, negative = angry/dissatisfied, neutral = neither"
    },
    workspace=workspace,
    created_by=user,
    approved=False  # Pending approval
)

# 2. Manager reviews and approves
sentiment_label.approved = True
sentiment_label.approved_by = manager
sentiment_label.save()

# 3. Link to projects
for project in sentiment_projects:
    LabelLink.objects.create(
        project=project,
        label=sentiment_label,
        from_name="sentiment"  # Matches label_config
    )

# 4. Update label affects all linked projects
sentiment_label.value["choices"].append("mixed")
sentiment_label.save()
# All projects now see "mixed" option
```

---

## 18. LabelLink

**Purpose:** Links label definitions to projects
**Table Name:** `label_link`
**Business Context:** Many-to-many relationship between labels and projects

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Link ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for label-project link |
| **project_id** | integer | Project reference | FK to project, CASCADE | **Project side** - Which project uses this label. CASCADE delete removes link when project deleted |
| **label_id** | integer | Label reference | FK to label, CASCADE | **Label side** - Which label is used. CASCADE delete removes link when label deleted |
| **from_name** | varchar(2048) | Control name in config | NOT NULL | **CRITICAL: Label config mapping** - Which control tag in project's label_config uses this label:<br>Example: `<Choices name="sentiment">` → from_name="sentiment"<br>Maps library label to specific control in UI |

### Business Logic Use Cases

**Primary Use Cases:**
- **Label Application:** Apply standard labels to new project
- **Bulk Updates:** Change label definition affects all linked projects
- **Label Tracking:** See which projects use which labels
- **Configuration Management:** Separate label values from project config

**Usage Example:**
```python
# Project's label_config:
label_config = """
<View>
    <Text name="review" value="$text"/>
    <Choices name="sentiment" toName="review" />
    <Choices name="urgency" toName="review" />
</View>
"""

# Link standard sentiment labels
LabelLink.objects.create(
    project=project,
    label=sentiment_label,  # choices: [positive, negative, neutral]
    from_name="sentiment"
)

# Link urgency labels
LabelLink.objects.create(
    project=project,
    label=urgency_label,  # choices: [low, medium, high, critical]
    from_name="urgency"
)

# Result: UI renders sentiment choices from sentiment_label
#         UI renders urgency choices from urgency_label
```

---

## SECTION 19: DATA MANAGER & VIEWS

---

## 19. data_manager_view

**Purpose:** Saved custom views in Data Manager (filters, column configs, selections)
**Table Name:** `data_manager_view`
**Business Context:** User-specific Data Manager configurations for quick access

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | View ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for saved view |
| **data** | jsonb | View configuration | NOT NULL | **View settings** - JSON containing Data Manager UI state:<br>- Column visibility (show/hide columns)<br>- Column widths<br>- Sort orders<br>- Display settings<br>Example: `{"columns": ["id", "data__image", "annotations"], "widths": {"id": 50, "data__image": 200}}` |
| **ordering** | jsonb | Sort configuration | NULL | **Sort order** - Array of column sort specs:<br>Example: `[{"column": "created_at", "order": "desc"}, {"column": "id", "order": "asc"}]`<br>Multi-column sorting support |
| **selected_items** | jsonb | Selected task IDs | NULL | **Selection state** - Array of task IDs selected in Data Manager:<br>Example: `[1, 5, 12, 18, 42]`<br>Persists selections between sessions |
| **filter_group_id** | integer | Associated filters | FK to FilterGroup, SET_NULL | **Filter configuration** - Links to FilterGroup containing complex filters. SET_NULL preserves view if filter deleted |
| **user_id** | integer | View owner | FK to htx_user, CASCADE | **User scope** - Which user owns this view. Views are private. CASCADE delete removes views when user deleted |
| **project_id** | integer | Parent project | FK to project, CASCADE | **Project scope** - Which project this view is for. CASCADE delete removes views when project deleted |

### Business Logic Use Cases

**Primary Use Cases:**
- **Custom Task Views:** Save filtered view of high-priority tasks
- **Column Presets:** Different column configurations for different workflows
- **Batch Operations:** Save selection of tasks for bulk actions
- **Personal Workflows:** Each user has their own saved views

**Saved View Workflow:**
```python
# 1. User customizes Data Manager:
# - Shows columns: ID, Image, Annotation Status, Created Date
# - Filters: Tasks with disagreement
# - Sorts: Created date descending
# - Selects: 10 tasks for review

# 2. User clicks "Save View"
view = data_manager_view.objects.create(
    user=user,
    project=project,
    data={
        "columns": ["id", "data__image", "annotations__status", "created_at"],
        "column_widths": {"id": 50, "data__image": 200, "annotations__status": 150, "created_at": 100}
    },
    ordering=[
        {"column": "created_at", "order": "desc"}
    ],
    selected_items=[42, 108, 203, 305, 417],
    filter_group=filter_group  # Linked FilterGroup with disagreement filter
)

# 3. User returns later, clicks saved view
# System restores: columns, widths, sort, filters, selections
# User continues where they left off
```

**Multiple Saved Views:**
```python
# User creates multiple views for different purposes
views = [
    {
        "name": "Needs Review",
        "data": {"columns": ["id", "review_status", "annotator"]},
        "filter": "review_status=pending"
    },
    {
        "name": "High Agreement",
        "data": {"columns": ["id", "agreement_score", "annotations"]},
        "filter": "agreement > 0.9"
    },
    {
        "name": "Recent Imports",
        "data": {"columns": ["id", "file_upload", "created_at"]},
        "filter": "created_at > 7 days ago"
    }
]
```

---

## 20. FilterGroup

**Purpose:** Container for complex filter groups with AND/OR logic
**Table Name:** `data_manager_filtergroup`
**Business Context:** Enables complex filtering: (condition1 AND condition2) OR (condition3 AND condition4)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Group ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for filter group |
| **conjunction** | varchar(1024) | Logical operator | NOT NULL | **Group logic** - How filters combined within group:<br>- "and": ALL filters must match<br>- "or": ANY filter must match<br>Example: Filter1 AND Filter2 AND Filter3 |

### Business Logic Use Cases

**Primary Use Cases:**
- **Complex Filters:** Combine multiple conditions with AND/OR logic
- **Nested Logic:** Groups can be nested for complex queries
- **Reusable Filters:** Save filter groups for reuse

**Filter Group Structure:**
```
FilterGroup (conjunction="and")
├─ Filter 1: annotation_status = "completed"
├─ Filter 2: created_at > "2024-01-01"
└─ Filter 3: agreement_score > 0.8

Result: Tasks that are completed AND created after Jan 1 AND have agreement > 0.8
```

**Nested Filter Groups:**
```
FilterGroup A (conjunction="or")
├─ FilterGroup B (conjunction="and")
│  ├─ Filter: language = "Spanish"
│  └─ Filter: domain = "Medical"
└─ FilterGroup C (conjunction="and")
   ├─ Filter: language = "French"
   └─ Filter: domain = "Legal"

Result: (Spanish AND Medical) OR (French AND Legal)
```

---

## 21. Filter

**Purpose:** Individual filter conditions for Data Manager queries
**Table Name:** `data_manager_filter`
**Business Context:** Represents single filter condition (e.g., "status = completed")

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Filter ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for filter |
| **index** | integer | Order in group | NOT NULL | **Display order** - Order of filter within FilterGroup (0, 1, 2...) |
| **column** | varchar(1024) | Column/field name | NOT NULL | **Filter target** - Which field to filter on:<br>- "id": Task ID<br>- "created_at": Creation date<br>- "data__image": Task data image field<br>- "annotations__status": Annotation status<br>Supports nested fields with __ notation |
| **type** | varchar(1024) | Column data type | NOT NULL | **Type hint** - Data type of column:<br>- "Number": Integer/float<br>- "String": Text<br>- "DateTime": Timestamp<br>- "Boolean": True/false<br>Determines available operators |
| **operator** | varchar(1024) | Comparison operator | NOT NULL | **Comparison type** - How to compare:<br>**Number/DateTime:** equal, not_equal, less, less_or_equal, greater, greater_or_equal, in_range, not_in_range<br>**String:** equal, not_equal, contains, not_contains, regex, empty, not_empty<br>**Boolean:** equal, not_equal |
| **value** | jsonb | Filter value(s) | NOT NULL | **Comparison value** - Value(s) to compare against:<br>- Single value: `{"value": "completed"}`<br>- Multiple values: `{"values": ["completed", "reviewed"]}`<br>- Range: `{"min": 0.8, "max": 1.0}`<br>JSON supports complex values |

### Business Logic Use Cases

**Primary Use Cases:**
- **Task Filtering:** Find tasks matching criteria
- **Quality Queries:** Filter by agreement score, review status
- **Date Ranges:** Tasks created/updated in timeframe
- **Complex Searches:** Multi-field searches with AND/OR logic

**Filter Examples:**

**Simple Filter:**
```python
Filter.objects.create(
    filter_group=group,
    index=0,
    column="annotations__status",
    type="String",
    operator="equal",
    value={"value": "completed"}
)
# Result: WHERE annotations.status = 'completed'
```

**Range Filter:**
```python
Filter.objects.create(
    filter_group=group,
    index=1,
    column="agreement_score",
    type="Number",
    operator="in_range",
    value={"min": 0.8, "max": 1.0}
)
# Result: WHERE agreement_score BETWEEN 0.8 AND 1.0
```

**Text Search:**
```python
Filter.objects.create(
    filter_group=group,
    index=2,
    column="data__text",
    type="String",
    operator="contains",
    value={"value": "medical"}
)
# Result: WHERE data->>'text' LIKE '%medical%'
```

**Date Range:**
```python
Filter.objects.create(
    filter_group=group,
    index=3,
    column="created_at",
    type="DateTime",
    operator="in_range",
    value={"min": "2024-01-01T00:00:00Z", "max": "2024-03-31T23:59:59Z"}
)
# Result: WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31'
```

**Multiple Values (IN operator):**
```python
Filter.objects.create(
    filter_group=group,
    index=4,
    column="task_status",
    type="String",
    operator="in",
    value={"values": ["pending", "in_progress", "review"]}
)
# Result: WHERE task_status IN ('pending', 'in_progress', 'review')
```

---

## SECTION 20: WEBHOOKS

---

## 22. webhook

**Purpose:** Real-time event notifications to external systems
**Table Name:** `webhook`
**Business Context:** Sends HTTP POST requests when events occur (task completed, annotation created, etc.)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Webhook ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for webhook |
| **workspace_id** | integer | Parent workspace | FK to workspace, SET_NULL | **Workspace scope** - Workspace-level webhooks (all projects). SET_NULL allows global webhooks |
| **project_id** | integer | Parent project | FK to project, SET_NULL | **Project scope** - Project-specific webhooks. SET_NULL if workspace-level. Can't have both workspace_id and project_id |
| **url** | varchar(2048) | Destination URL | NOT NULL | **CRITICAL: Endpoint** - Where to send HTTP POST:<br>Example: "https://api.company.com/webhooks/labelstudio"<br>Must be HTTPS in production for security |
| **send_payload** | boolean | Include event data | DEFAULT TRUE | **Payload control** - TRUE: Send full event data in POST body. FALSE: Just send notification, recipient must fetch data via API |
| **send_for_all_actions** | boolean | All events flag | DEFAULT TRUE | **Event filter** - TRUE: Send for all events. FALSE: Only send for events in webhook_action table |
| **headers** | jsonb | HTTP headers | NULL | **Authentication & config** - Custom HTTP headers for webhook request:<br>Example: `{"Authorization": "Bearer secret_token", "X-Custom-Header": "value"}`<br>Used for auth, routing, signatures |
| **is_active** | boolean | Webhook enabled | DEFAULT TRUE | **Toggle** - TRUE: Webhook fires on events. FALSE: Disabled without deleting. Useful for debugging/maintenance |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Webhook creation time** |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - When webhook config changed |

### Business Logic Use Cases

**Primary Use Cases:**
- **Integration:** Notify external systems of Label Studio events
- **Workflow Automation:** Trigger downstream processes when tasks complete
- **Monitoring:** Send events to logging/monitoring systems
- **Custom Workflows:** Drive custom business logic based on annotations

**Common Webhook Scenarios:**

**Task Completion Notification:**
```python
Webhook.objects.create(
    project=project,
    url="https://api.company.com/webhooks/task-completed",
    send_payload=True,
    send_for_all_actions=False,  # Only specific actions
    headers={
        "Authorization": "Bearer sk_webhook_secret_abc123",
        "X-Project-ID": str(project.id)
    },
    is_active=True
)

# Add specific action
webhook_action.objects.create(
    webhook=webhook,
    action="ANNOTATION_CREATED"
)

# Event fires: POST to URL with payload:
{
    "event": "ANNOTATION_CREATED",
    "project": 123,
    "task": 456,
    "annotation": {
        "id": 789,
        "completed_by": {"id": 42, "email": "annotator@company.com"},
        "result": [...]
    }
}
```

**Quality Monitoring:**
```python
Webhook.objects.create(
    workspace=workspace,
    url="https://monitoring.company.com/labelstudio/events",
    send_payload=True,
    send_for_all_actions=True,  # All events
    headers={
        "X-Signature": "hmac_signature_here",
        "X-Workspace-ID": str(workspace.id)
    }
)

# All events sent: task created, annotation created, review completed, etc.
```

**Webhook Security:**
```python
# 1. HMAC Signature
import hmac
import hashlib

def sign_payload(payload, secret):
    signature = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return signature

webhook_secret = "webhook_secret_key"
payload = json.dumps(event_data)
signature = sign_payload(payload, webhook_secret)

headers = {
    "X-Signature": signature,
    "X-Timestamp": str(int(time.time()))
}

# 2. Recipient verifies signature
def verify_webhook(request):
    signature = request.headers.get("X-Signature")
    timestamp = request.headers.get("X-Timestamp")

    # Check timestamp (prevent replay attacks)
    if abs(int(time.time()) - int(timestamp)) > 300:  # 5 minutes
        return False

    # Verify signature
    expected_sig = sign_payload(request.body, webhook_secret)
    return hmac.compare_digest(signature, expected_sig)
```

---

## 23. webhook_action

**Purpose:** Specific event types that trigger webhook
**Table Name:** `webhook_action`
**Business Context:** Fine-grained control over which events fire webhook

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Action ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for webhook action |
| **webhook_id** | integer | Parent webhook | FK to webhook, CASCADE | **Webhook reference** - Which webhook this action belongs to. CASCADE delete removes actions when webhook deleted |
| **action** | varchar(128) | Event type | NOT NULL | **Event trigger** - Which event fires webhook:<br>**Task Events:** TASK_CREATED, TASK_UPDATED, TASK_DELETED<br>**Annotation Events:** ANNOTATION_CREATED, ANNOTATION_UPDATED, ANNOTATION_DELETED<br>**Review Events:** ANNOTATION_REVIEWED, ANNOTATION_ACCEPTED, ANNOTATION_REJECTED<br>**Project Events:** PROJECT_UPDATED<br>**Import Events:** IMPORT_COMPLETED, IMPORT_FAILED |

### Business Logic Use Cases

**Primary Use Cases:**
- **Event Filtering:** Only send specific events to endpoint
- **Multi-webhook Setup:** Different endpoints for different events
- **Reduced Noise:** Avoid overwhelming external system with all events

**Filtered Webhook Example:**
```python
# Webhook only fires for annotation review events
webhook = Webhook.objects.create(
    project=project,
    url="https://qa.company.com/review-notifications",
    send_for_all_actions=False  # Must be False to use webhook_action
)

# Add specific actions
for action in ["ANNOTATION_REVIEWED", "ANNOTATION_ACCEPTED", "ANNOTATION_REJECTED"]:
    webhook_action.objects.create(
        webhook=webhook,
        action=action
    )

# Result: Webhook fires only when:
# - Annotation is reviewed
# - Annotation is accepted
# - Annotation is rejected
# Does NOT fire for: task created, annotation created, etc.
```

**Multi-webhook Architecture:**
```python
# Webhook 1: Task lifecycle → Task tracking system
task_webhook = Webhook.objects.create(
    project=project,
    url="https://tasks.company.com/webhook",
    send_for_all_actions=False
)
for action in ["TASK_CREATED", "TASK_UPDATED", "TASK_DELETED"]:
    webhook_action.objects.create(webhook=task_webhook, action=action)

# Webhook 2: Annotations → ML training pipeline
ml_webhook = Webhook.objects.create(
    project=project,
    url="https://ml.company.com/webhook",
    send_for_all_actions=False
)
for action in ["ANNOTATION_CREATED", "ANNOTATION_ACCEPTED"]:
    webhook_action.objects.create(webhook=ml_webhook, action=action)

# Webhook 3: Quality metrics → Analytics system
qa_webhook = Webhook.objects.create(
    project=project,
    url="https://analytics.company.com/webhook",
    send_for_all_actions=False
)
for action in ["ANNOTATION_REVIEWED", "ANNOTATION_ACCEPTED", "ANNOTATION_REJECTED"]:
    webhook_action.objects.create(webhook=qa_webhook, action=action)
```

---

## 🔄 BUSINESS WORKFLOWS & DATA FLOWS

### Workflow 1: Creating New Project with Labeling Interface

```
Step 1: Create Project
project = Project.objects.create(
    title="Sentiment Analysis",
    description="Classify customer reviews by sentiment",
    workspace=workspace,
    created_by=user
)

Step 2: Configure Labeling Interface
project.label_config = """
<View>
    <Text name="review" value="$review_text"/>
    <Choices name="sentiment" toName="review" choice="single">
        <Choice value="positive"/>
        <Choice value="negative"/>
        <Choice value="neutral"/>
    </Choices>
</View>
"""
project.save()  # Auto-generates parsed_label_config

Step 3: Set Quality Control
project.maximum_annotations = 2  # 2 annotations per task
project.review_percentage = 20   # 20% go to review
project.overlap_cohort_percentage = 100  # All tasks get overlap
project.save()

Step 4: Configure Task Distribution
ProjectOpsManager.objects.create(
    project=project,
    split_strategy="EQUAL",
    max_limit=0
)

Step 5: Add Team Members
for annotator in annotators:
    ProjectMember.objects.create(
        project=project,
        user=annotator,
        member_type=ANNOTATOR,
        task_limit=0  # Unlimited
    )

Step 6: Publish Project
project.is_published = True
project.state = "live"
project.save()
```

### Workflow 2: Importing Tasks with Pre-annotations

```
Step 1: User Uploads CSV
CSV format:
review_text,prediction
"Great product!",{"sentiment": "positive", "score": 0.95}
"Terrible service.",{"sentiment": "negative", "score": 0.88}

Step 2: Create FileUpload
file_upload = FileUpload.objects.create(
    user=user,
    project=project,
    file=csv_file
)

Step 3: Create Import Job
import_job = ProjectImport.objects.create(
    project=project,
    preannotated_from_fields=["prediction"],  # Use prediction as pre-annotation
    commit_to_project=True,
    return_task_ids=True,
    status="created",
    file_upload_ids=[file_upload.id]
)

Step 4: Background Worker Processes
def process_import(import_job):
    rows = parse_csv(file_upload.file)

    for row in rows:
        # Create task
        task = Task.objects.create(
            project=project,
            data={"review_text": row["review_text"]}
        )

        # Create pre-annotation from prediction
        if "prediction" in row:
            Annotation.objects.create(
                task=task,
                project=project,
                result=[{"value": row["prediction"]}],
                was_cancelled=False
            )

        import_job.task_count += 1
        import_job.annotation_count += 1 if "prediction" in row else 0

    import_job.status = "completed"
    import_job.save()
```

### Workflow 3: Setting Up Webhooks for Integration

```
Step 1: Create Webhook
webhook = Webhook.objects.create(
    project=project,
    url="https://api.company.com/labelstudio/events",
    send_payload=True,
    send_for_all_actions=False,
    headers={
        "Authorization": "Bearer secret_token",
        "X-Project-ID": str(project.id)
    },
    is_active=True
)

Step 2: Configure Event Filters
webhook_action.objects.create(webhook=webhook, action="ANNOTATION_CREATED")
webhook_action.objects.create(webhook=webhook, action="ANNOTATION_REVIEWED")

Step 3: Event Fires
# When annotation created:
send_webhook(
    url=webhook.url,
    headers=webhook.headers,
    payload={
        "event": "ANNOTATION_CREATED",
        "project_id": project.id,
        "task_id": task.id,
        "annotation_id": annotation.id,
        "annotation_data": annotation.result
    }
)

Step 4: External System Receives
# POST https://api.company.com/labelstudio/events
# Headers: Authorization: Bearer secret_token
# Body: {event data}

# External system processes:
def handle_webhook(request):
    event = request.json
    if event["event"] == "ANNOTATION_CREATED":
        # Update downstream systems
        # Trigger ML retraining
        # Send notifications
        pass
```

### Workflow 4: Creating Project with Label Library

```
Step 1: Create Standard Labels in Workspace
sentiment_label = Label.objects.create(
    workspace=workspace,
    title="Standard Sentiment Labels",
    value={"choices": ["positive", "negative", "neutral"]},
    description="Company-wide sentiment labels",
    approved=True,
    approved_by=manager
)

Step 2: Create Project with Label Config
project = Project.objects.create(
    title="Customer Reviews",
    workspace=workspace,
    label_config="""
    <View>
        <Text name="review" value="$review_text"/>
        <Choices name="sentiment" toName="review"/>
    </View>
    """
)

Step 3: Link Label to Project
LabelLink.objects.create(
    project=project,
    label=sentiment_label,
    from_name="sentiment"  # Matches <Choices name="sentiment"/>
)

Step 4: Label Update Propagates
# Manager updates label
sentiment_label.value["choices"].append("mixed")
sentiment_label.save()

# All projects using this label now have "mixed" option
# No need to update each project's label_config
```

### Workflow 5: Saved Data Manager Views

```python
Step 1: User Creates Custom View
# User in Data Manager:
# - Filters: Tasks with low agreement (<0.7)
# - Columns: ID, Text, Agreement Score, Annotations
# - Sort: Agreement score ascending (worst first)

Step 2: Create Filter Group
filter_group = FilterGroup.objects.create(conjunction="and")

Filter.objects.create(
    filter_group=filter_group,
    index=0,
    column="agreement_score",
    type="Number",
    operator="less",
    value={"value": 0.7}
)

Step 3: Save the View
view = DataManagerView.objects.create(
    project=project,
    user=reviewer,
    filter_group=filter_group,
    data={
        "columnsWidth": {},
        "columnsDisplayType": {},
        "gridWidth": 4,
        "ordering": ["-agreement_score"],  # Worst first
        "selectedItems": {}
    }
)

Step 4: Share View with Team
view.shared = True
view.save()

# Now all team members see "Low Agreement Tasks" in saved views dropdown
# When clicked, data manager automatically applies filters and sorting
```

---

## 🔧 TECHNICAL IMPLEMENTATION NOTES

### Performance Considerations

**Project Table Optimization:**
- `is_published` and `created_at` are heavily indexed for project listing queries
- `task_number` and `task_data_login` use database sequences for lock-free increments
- `parsed_label_config` caches XML parsing results to avoid repeated parsing overhead
- `num_tasks_with_annotations` enables fast "completion percentage" calculations without joins

**ProjectSummary Table:**
- Acts as materialized view for expensive aggregation queries
- Updated via triggers or background jobs when annotations change
- Critical for dashboard performance with 1000+ task projects

**Data Manager Views:**
- Filter JSON stored denormalized for fast view loading
- Complex filters (regex, date ranges) cached in filter_group
- `index` field ensures consistent filter evaluation order

**Webhook Performance:**
- `is_active` filter applied first to skip disabled webhooks
- `send_for_all_actions` short-circuits action checking
- Failed webhooks marked inactive after retry threshold

### Common Pitfalls & Gotchas

**1. Label Config XML Parsing:**
```python
# WRONG: Directly accessing control tags
label_config = project.label_config  # Raw XML string
# This requires parsing every time

# RIGHT: Use parsed_label_config
parsed = project.parsed_label_config  # Already parsed dict
from_names = parsed.get("from_names", [])
```

**2. Project Member Permissions:**
```python
# WRONG: Checking only ProjectMember
if ProjectMember.objects.filter(project=project, user=user).exists():
    # Missing workspace-level and org-level permissions

# RIGHT: Check full permission hierarchy
if user.has_project_permission(project):
    # Checks ProjectMember, WorkspaceMember, OrganizationMember
```

**3. Webhook Action Filtering:**
```python
# WRONG: Firing webhook for every action
webhook.fire(action="ANNOTATION_CREATED", payload=data)

# RIGHT: Check if webhook listens to this action
if webhook.send_for_all_actions or \
   WebhookAction.objects.filter(webhook=webhook, action="ANNOTATION_CREATED").exists():
    webhook.fire(action="ANNOTATION_CREATED", payload=data)
```

**4. ProjectImport Status Tracking:**
```python
# WRONG: Assuming import is done when created
import_obj = ProjectImport.objects.get(id=import_id)
tasks = Task.objects.filter(project=import_obj.project)  # May be incomplete

# RIGHT: Check status and completion
import_obj = ProjectImport.objects.get(id=import_id)
if import_obj.status == "completed":
    # Safe to query tasks
```

**5. Filter Group Conjunction Logic:**
```python
# Filter conjunction determines AND vs OR logic
filter_group = FilterGroup.objects.get(id=fg_id)

if filter_group.conjunction == "and":
    # ALL filters must match
    query = Q()
    for f in filter_group.filters.all():
        query &= build_filter_q(f)
else:  # "or"
    # ANY filter can match
    query = Q()
    for f in filter_group.filters.all():
        query |= build_filter_q(f)
```

### Integration Points with Other Systems

**External Storage (Section 22-26):**
- `FileUpload.storage` references external storage configurations
- Projects can specify default storage for imported files
- Storage type affects URL generation and file access patterns

**Tasks (Section 11):**
- `project.task_number` tracks next task ID
- `project.maximum_annotations` limits annotations per task
- `project.show_overlap_first` affects task distribution

**Annotations (Section 12):**
- `project.expert_instruction` shown to annotators
- `project.control_weights` configures honeypot tasks
- `project.reveal_preannotations_interactively` affects ML prediction display

**ML Backends (Section 14):**
- `project.model_version` tracks active ML model
- Projects trigger ML training via background jobs
- `ProjectEmbedding` stores vector representations for semantic search

**Organizations & Workspaces (Sections 4, 6-7):**
- Projects belong to workspaces, not directly to organizations
- Workspace-level billing affects project creation limits
- Organization-level settings (like `contact_info`) inherited by projects

### Testing Checklist

**Project Creation:**
- [ ] Valid label_config XML parses without errors
- [ ] parsed_label_config generated correctly
- [ ] Default values applied (color, sampling, etc.)
- [ ] Workspace membership verified
- [ ] Organization limits not exceeded

**Project Configuration:**
- [ ] ProjectClientApiConfiguration TTS/translation work
- [ ] Onboarding steps displayed in correct order
- [ ] Tags properly linked via projects_tag junction
- [ ] Member permissions cascade correctly

**Data Import:**
- [ ] ProjectImport handles CSV, JSON, TSV formats
- [ ] FileUpload validates file types and sizes
- [ ] Import status tracked accurately (in_progress → completed/failed)
- [ ] Tasks created with correct data references
- [ ] Duplicate detection works (ProjectReimport)

**Data Manager Views:**
- [ ] FilterGroups with multiple filters combine correctly (AND/OR)
- [ ] Filter operators work (equals, contains, greater, etc.)
- [ ] Saved views load filter state properly
- [ ] Shared views visible to all project members

**Webhooks:**
- [ ] Webhook URLs reachable and return 2xx status
- [ ] Webhook headers applied correctly
- [ ] Actions filter working (send_for_all_actions vs specific)
- [ ] Payload includes all required fields
- [ ] Failed webhooks auto-disabled after retries

**Labels:**
- [ ] LabelLink connects labels to correct from_name controls
- [ ] Label updates propagate to all linked projects
- [ ] Label deletion handled gracefully

---

## 📊 KEY METRICS & MONITORING

### Project Health Metrics

1. **Completion Rate:** `num_tasks_with_annotations / total_tasks * 100`
2. **Average Annotation Time:** Tracked in ProjectSummary
3. **Active Members:** ProjectMember count with recent activity
4. **Storage Usage:** Sum of FileUpload sizes per project
5. **Webhook Success Rate:** Successful fires / total attempts

### Performance Indicators

- **Label Config Parse Time:** Should be <50ms for cached configs
- **Data Manager View Load Time:** <200ms for views with <5 filters
- **Import Speed:** >100 tasks/second for simple JSON imports
- **Webhook Delivery Time:** <5 seconds for 95th percentile

### Alert Triggers

- ProjectImport stuck in "in_progress" >1 hour
- Webhook failure rate >10%
- FileUpload success rate <95%
- ProjectSummary not updated in >24 hours
- DataManagerView queries timing out

---

## 🚀 ADVANCED PATTERNS

### Dynamic Label Configuration

Projects support runtime label config modification:

```python
# Add new choice to existing config
project = Project.objects.get(id=123)
config = project.parsed_label_config

# Append new choice
config["choices"]["sentiment"].append("confused")

# Regenerate XML and save
project.label_config = render_label_config_xml(config)
project.save()  # Triggers parsed_label_config regeneration
```

### Cascading Project Templates

```python
# Create template project
template = Project.objects.create(
    title="[TEMPLATE] Image Classification",
    is_published=False,  # Hidden from regular users
    label_config=STANDARD_IMAGE_CONFIG,
    expert_instruction="Standard classification guidelines...",
)

# Clone from template
new_project = Project.objects.create(
    title="Real Project - Batch 5",
    workspace=workspace,
    label_config=template.label_config,
    expert_instruction=template.expert_instruction,
    sampling=template.sampling,
    show_instruction=template.show_instruction,
)
```

### Complex Filter Combinations

```python
# Filter: (status=completed AND agreement<0.7) OR (status=skipped)
group1 = FilterGroup.objects.create(conjunction="and")
Filter.objects.create(group=group1, column="status", operator="equal", value={"value": "completed"})
Filter.objects.create(group=group1, column="agreement", operator="less", value={"value": 0.7})

group2 = FilterGroup.objects.create(conjunction="and")
Filter.objects.create(group=group2, column="status", operator="equal", value={"value": "skipped"})

# Combine groups with OR
root_group = FilterGroup.objects.create(conjunction="or")
root_group.children.add(group1, group2)

# Use in view
view = DataManagerView.objects.create(
    project=project,
    filter_group=root_group,
)
```

---
# PROJECTS & CONFIGURATION - COMPREHENSIVE ENHANCEMENTS
## Additional Documentation Sections

---

## 🎯 SYSTEM OVERVIEW

### What Projects Do in the Big Picture

The **Project** is the central orchestration entity in the Label Studio platform. It serves as the complete workspace for annotation workflows, from raw data ingestion to quality-controlled labeled datasets.

**Core Functions:**
1. **Data Container**: Holds all tasks (items to be annotated) within a workspace scope
2. **Interface Definition**: XML-based label_config defines custom annotation UIs
3. **Workflow Orchestrator**: Controls task distribution, quality control, and review processes
4. **Team Coordination**: Manages annotators, reviewers, and administrators with role-based access
5. **Integration Hub**: Connects ML backends, external APIs, webhooks, and export pipelines
6. **Quality Controller**: Enforces overlap requirements, review percentages, and golden reference validation

**Position in Platform Hierarchy:**
```
Organization
  └── Workspace (data isolation boundary)
       └── Project (annotation workspace)
            ├── Tasks (items to annotate)
            │    ├── Annotations (labeling results)
            │    └── Predictions (ML pre-annotations)
            ├── ProjectMembers (team assignments)
            ├── DataManagerViews (saved filters)
            ├── Webhooks (event notifications)
            └── ProjectImports/Exports (data pipelines)
```

**Lifecycle States:**
- **PENDING**: Created but not configured/ready
- **BUILDING_WORKFLOW**: Configuration in progress
- **LIVE**: Active annotation work happening
- **PAUSED**: Temporarily stopped (team changes, config updates)
- **COMPLETED**: All tasks annotated to satisfaction

**Key Design Philosophy:**
- **Workspace Isolation**: All project data scoped to workspace (CASCADE delete protection)
- **Configuration as Data**: Label configs stored as XML, validated against existing annotations
- **Eventual Consistency**: ProjectSummary caches expensive aggregations, updated via signals
- **Flexible Quality Control**: Overlap, review, and golden reference percentages configurable per project
- **Multi-stage Support**: ProjectNestedRelations enable review/adjudication pipelines

---

## 👥 USER PERSONAS AND USE CASES

### Persona 1: Project Manager (Sarah)

**Role**: Sets up and manages annotation projects
**Goals**: Fast project setup, monitor progress, ensure quality
**Pain Points**: Complex configuration, unclear progress tracking

**Primary Use Cases:**
1. **Create New Project** (Weekly)
   - Clones from template project
   - Customizes label_config for new data type
   - Sets maximum_annotations=2 for quality control

2. **Monitor Project Progress** (Daily)
   - Checks ProjectSummary for annotation counts
   - Reviews agreement metrics from control_weights
   - Identifies slow annotators via ProjectMember task counts

3. **Quality Control Setup** (Per Project)
   - Sets review_percentage=20 for spot-checking
   - Configures golden_ref_percentage=5 for benchmarking
   - Creates nested review project for rejected annotations

**Typical SQL Queries:**
```sql
-- Project progress overview
SELECT
    p.id,
    p.title,
    p.state,
    COUNT(t.id) as total_tasks,
    COUNT(CASE WHEN t.is_labeled THEN 1 END) as completed_tasks,
    ROUND(100.0 * COUNT(CASE WHEN t.is_labeled THEN 1 END) / NULLIF(COUNT(t.id), 0), 2) as progress_pct
FROM project p
LEFT JOIN task t ON t.project_id = p.id
WHERE p.workspace_id = 123 AND p.is_deleted = FALSE
GROUP BY p.id, p.title, p.state;
```

### Persona 2: Annotator (Mike)

**Role**: Labels tasks according to project guidelines
**Goals**: Clear instructions, efficient task flow, avoid rework
**Pain Points**: Confusing UIs, unclear edge cases, getting same task twice

**Primary Use Cases:**
1. **Get Next Task** (Hundreds per day)
   - System uses sampling strategy (SEQUENCE/UNIFORM/UNCERTAINTY)
   - ProjectOpsManager enforces task limits per annotator
   - Skips locked tasks (collaborative annotation)

2. **Submit Annotation** (After each task)
   - Creates Annotation record linked to task
   - Updates ProjectSummary counters via signals
   - Triggers webhook if configured (ANNOTATION_CREATED)
   - Checks if task.is_labeled should flip (reached maximum_annotations)

3. **Skip Difficult Task** (Occasionally)
   - Behavior controlled by project.skip_queue:
     - REQUEUE_FOR_ME: Goes back to end of Mike's queue
     - REQUEUE_FOR_OTHERS: Goes to other annotators
     - IGNORE_SKIPPED: Counts as completion

**Task Assignment Logic:**
```
get_next_task(project, user):
  1. Check task limits (ProjectOpsManager.get_limit)
  2. If review_percentage > 0 and user is reviewer:
       return get_next_review_task()
  3. If golden_ref_percentage > 0 and random < percentage:
       return ground truth task
  4. Filter: project tasks NOT annotated by user AND NOT locked
  5. Apply sampling:
       - SEQUENCE: ORDER BY data manager sorting
       - UNIFORM: ORDER BY random()
       - UNCERTAINTY: ORDER BY prediction uncertainty DESC
  6. If show_overlap_first: prioritize tasks with overlap > 1
  7. If show_ground_truth_first: prioritize tasks with ground_truth annotations
  8. Return first unlocked task
```

### Persona 3: ML Engineer (Priya)

**Role**: Integrates ML models for pre-annotations and active learning
**Goals**: Provide helpful predictions, trigger training, measure model quality
**Pain Points**: Predictions not displaying, training triggers unclear

**Primary Use Cases:**
1. **Configure ML Backend** (Per Project)
   - Adds ML backend URL to project settings
   - Sets model_version to track predictions
   - Enables show_collab_predictions for annotators to see predictions

2. **Submit Predictions** (Batch)
   - Creates Prediction records for tasks
   - Sets model_version to identify prediction source
   - Predictions shown to annotators if show_collab_predictions=True

3. **Active Learning** (Automated)
   - Sets sampling="UNCERTAINTY"
   - Predictions include uncertainty scores
   - High-uncertainty tasks prioritized for annotation
   - Triggers training when min_annotations_to_start_training reached

**Prediction Workflow:**
```python
# ML backend submits predictions
for task in tasks:
    Prediction.objects.create(
        task=task,
        model_version="sentiment-v2.3",
        result=[{"value": {"choices": ["positive"]}, "score": 0.87}]
    )

# Project uses predictions
project.model_version = "sentiment-v2.3"
project.sampling = "UNCERTAINTY"
project.show_collab_predictions = True
project.save()

# Annotators see predictions in UI
# High uncertainty tasks (score < 0.6) assigned first
```

### Persona 4: Quality Analyst (James)

**Role**: Reviews annotations for quality and consistency
**Goals**: Find low-quality work, provide feedback, maintain standards
**Pain Points**: Hard to find problematic annotations, limited feedback mechanisms

**Primary Use Cases:**
1. **Review Queue Management** (Daily)
   - Receives review_percentage of completed tasks
   - Uses DataManagerView filters to find low-agreement tasks
   - Reviews annotations in nested review project

2. **Agreement Analysis** (Weekly)
   - Queries annotations with control_weights for agreement scores
   - Identifies annotators with consistently low agreement
   - Flags tasks for rework or adjudication

3. **Golden Reference Validation** (Per Project Setup)
   - Creates ground_truth annotations for golden_ref_percentage of tasks
   - Monitors annotator accuracy against golden references
   - Updates expert_instruction based on common mistakes

**Quality Queries:**
```sql
-- Find low agreement tasks
SELECT
    t.id,
    t.data,
    COUNT(a.id) as annotation_count,
    AVG(a.result::jsonb->0->'value'->>'agreement') as avg_agreement
FROM task t
JOIN task_completion a ON a.task_id = t.id
WHERE t.project_id = 456
  AND a.was_cancelled = FALSE
GROUP BY t.id
HAVING COUNT(a.id) >= 2 AND AVG((a.result::jsonb->0->'value'->>'agreement')::float) < 0.7
ORDER BY avg_agreement ASC;
```

---

## 🔄 COMPLETE ENUMERATION BUSINESS LOGIC

### 1. ProjectState Enumeration

**Source:** `label_studio/projects/models.py:141-146`

| State | Value | Business Logic | When Used |
|-------|-------|----------------|-----------|
| **PENDING** | "pending" | Project created but not ready for annotation | Default state on creation. Missing label_config or team members |
| **BUILDING_WORKFLOW** | "building_workflow" | Configuration in progress | Admin actively setting up label_config, importing initial tasks, configuring quality controls |
| **LIVE** | "live" | Active annotation work happening | Tasks available, annotators assigned, ready for production work |
| **PAUSED** | "paused" | Temporarily stopped | Team changes, label_config updates, or deliberate pause. Tasks exist but not assignable |
| **COMPLETED** | "completed" | All required annotations finished | All tasks reach maximum_annotations or manually marked complete |

**State Transitions:**
```
PENDING → BUILDING_WORKFLOW (admin starts configuration)
BUILDING_WORKFLOW → LIVE (admin publishes: is_published=True)
LIVE ⇄ PAUSED (admin pauses/resumes)
LIVE → COMPLETED (all tasks finished OR manual completion)
COMPLETED → LIVE (admin reopens for more annotations)
```

**Business Rules:**
- Only LIVE projects appear in annotator task queues
- State changes tracked via responsible_person_id
- Pausing doesn't delete data, just hides from annotators
- COMPLETED state is soft—can be reverted

### 2. ProjectType Enumeration

**Source:** `label_studio/projects/models.py:148-152`

| Type | Value | Business Logic | Use Case |
|------|-------|----------------|----------|
| **LABELLING** | "labelling" | Standard annotation project | General-purpose data labeling.|
| **ASSESSMENT** | "assessment" | Quality assessment workflow | Evaluates existing annotators.|
| **DP_ASSESSMENT** | "dp_assessment" | Data production assessment | Custom workflow for assessing data production quality.|
| **DP_ORDER** | "dp_order" | Data production order | Commissioned data collection/annotation orders. Tracked separately for billing |
|**DOUBLE_LABELLING**|"double_labelling"||For the number of annotators will be checking the same task|

### 3. Sampling Strategy Enumeration

**Source:** `label_studio/projects/models.py:314-324`

| Strategy | Value | Business Logic | Algorithm |
|----------|-------|----------------|-----------|
| **SEQUENCE** | "Sequential sampling" | Tasks ordered by Data Manager sorting | Default ORDER BY from DataManagerView. Predictable, deterministic assignment |
| **UNIFORM** | "Uniform sampling" | Random task selection | `ORDER BY random()`. Evenly distributes work, prevents cherry-picking |
| **UNCERTAINTY** | "Uncertainty sampling" | Active learning mode | `ORDER BY prediction_score ASC`. Prioritizes tasks where ML model is least confident |
|**EQUAL_DISTRIBUTION**|"Equally distributed blind pass sampling"|Tasks are distributed equally between users for balanced blind-pass and cross-check work|

**Detailed Algorithm (next_task.py:429-562):**

**SEQUENCE (Default):**
```python
def get_next_task_sequence(project, user):
    tasks = project.tasks.filter(
        ~Q(annotations__completed_by=user),  # Not annotated by me
        is_labeled=False  # Not completed
    ).order_by(*get_data_manager_ordering(project))

    return _get_first_unlocked(tasks, user)
```

**UNIFORM:**
```python
def get_next_task_uniform(project, user):
    tasks = project.tasks.filter(
        ~Q(annotations__completed_by=user),
        is_labeled=False
    )

    # Random sampling with lock-skip optimization
    return _get_random_unlocked(tasks, user)
```

**UNCERTAINTY:**
```python
def get_next_task_uncertainty(project, user):
    # Prioritize tasks with ML predictions having low confidence
    tasks = project.tasks.filter(
        ~Q(annotations__completed_by=user),
        is_labeled=False,
        predictions__model_version=project.model_version
    ).annotate(
        uncertainty=1.0 - Max('predictions__score')
    ).order_by('-uncertainty')  # High uncertainty first

    return _get_first_unlocked(tasks, user)
```

**Business Impact:**
- **SEQUENCE**: Best for maintaining context (e.g., annotating video frames in order)
- **UNIFORM**: Best for preventing bias (annotators don't pick easy tasks)
- **UNCERTAINTY**: Best for ML efficiency (focus human effort where model struggles)

### 4. SkipQueue Enumeration

**Source:** `label_studio/projects/models.py:133-140`

| Strategy | Value | Business Logic | When Task Skipped |
|----------|-------|----------------|-------------------|
| **REQUEUE_FOR_ME** | "REQUEUE_FOR_ME" | Skip goes back to same annotator's queue | Task added to END of annotator's queue. They'll see it again after other tasks |
| **REQUEUE_FOR_OTHERS** | "REQUEUE_FOR_OTHERS" | Skip goes to other annotators (DEFAULT) | Task excluded from skipping annotator's future queries, assigned to others |
| **IGNORE_SKIPPED** | "IGNORE_SKIPPED" | Skip counts as completion | Task marked as if annotated. Skipped annotation saved with was_cancelled=True |

**Implementation (next_task.py:32-51):**
```python
# REQUEUE_FOR_ME
if project.skip_queue == "REQUEUE_FOR_ME":
    # Task remains in user's queue
    tasks = tasks.filter(~Q(annotations__completed_by=user, annotations__was_cancelled=True))
    # Returns task to user eventually

# REQUEUE_FOR_OTHERS
elif project.skip_queue == "REQUEUE_FOR_OTHERS":
    # Permanently exclude from skipping user
    tasks = tasks.filter(~Q(annotations__completed_by=user))

# IGNORE_SKIPPED
elif project.skip_queue == "IGNORE_SKIPPED":
    # Skip creates cancelled annotation
    Annotation.objects.create(
        task=task,
        completed_by=user,
        was_cancelled=True,
        result=[]
    )
    # Counts toward maximum_annotations
    if task.annotations.filter(was_cancelled=False).count() >= project.maximum_annotations:
        task.is_labeled = True
        task.save()
```

**Use Case Guidance:**
- **REQUEUE_FOR_ME**: Training scenarios (annotator will have more context later)
- **REQUEUE_FOR_OTHERS**: Production (different annotator might find task easier)
- **IGNORE_SKIPPED**: Time-sensitive projects (skips count as "no label needed")

### 5. TaskSplitStrategy Enumeration

**Source:** `label_studio/projects/models.py:1586-1622` (ProjectOpsManager)

| Strategy | Value | Business Logic | Limit Calculation |
|----------|-------|----------------|-------------------|
| **EQUAL** | "EQUAL" | Divide tasks equally among annotators | `total_tasks / num_annotators` |
| **MAX_LIMIT** | "MAX_LIMIT" | Hard cap on tasks per annotator | `max_limit` (configured value) |
| **USER_LIMITS** | "USER_LIMITS" | Individual limits per user | `ProjectMember.task_limit` |
| **ORGANIZATION_LIMITS** | "ORGANIZATION_LIMITS" | Organization-level caps | `OrganizationMember.task_limit` |

**Implementation:**
```python
class ProjectOpsManager(models.Model):
    split_strategy = models.CharField(choices=SPLIT_STRATEGIES)
    max_limit = models.IntegerField(default=0)  # 0 = unlimited

    def get_limit(self, user):
        if self.split_strategy == "EQUAL":
            total = self.project.tasks.count()
            annotators = ProjectMember.objects.filter(
                project=self.project,
                member_type=MemberTypes.ANNOTATOR
            ).count()
            return total // annotators if annotators > 0 else 0

        elif self.split_strategy == "MAX_LIMIT":
            return self.max_limit

        elif self.split_strategy == "USER_LIMITS":
            member = ProjectMember.objects.get(project=self.project, user=user)
            return member.task_limit

        elif self.split_strategy == "ORGANIZATION_LIMITS":
            org_member = OrganizationMember.objects.get(
                organization=self.project.workspace.organization,
                user=user
            )
            return org_member.task_limit
```

**Business Rules:**
- Enforced in `get_next_task()` before assigning tasks
- 0 always means unlimited
- USER_LIMITS override MAX_LIMIT
- Used for: workload balancing, billing limits, SLA enforcement

### 6. MemberTypes Enumeration

**Source:** `core/utils/member_type.py`

| Role | Value | Permissions | Capabilities |
|------|-------|-------------|--------------|
| **ADMIN** | 1 | Full project control | Edit config, manage members, delete project, view all data |
| **ADJUDICATOR** | 2 | Resolve conflicts | View disagreements, create authoritative annotations |
| **REVIEWER** | 3 | QA annotations | See review queue (review_percentage), accept/reject annotations |
| **ANNOTATOR** | 4 | Label tasks | Get assigned tasks, submit annotations, view own work |

**Access Control Matrix:**
| Action | ADMIN | ADJUDICATOR | REVIEWER | ANNOTATOR |
|--------|-------|-------------|----------|-----------|
| Edit label_config | ✅ | ❌ | ❌ | ❌ |
| Add/remove members | ✅ | ❌ | ❌ | ❌ |
| View all annotations | ✅ | ✅ | ✅ | ❌ (own only) |
| Delete annotations | ✅ | ✅ | ✅ (in review queue) | ❌ |
| Get next task | ✅ | ✅ | ✅ | ✅ |
| Submit annotation | ✅ | ✅ | ✅ | ✅ |
| Access review queue | ✅ | ✅ | ✅ | ❌ |
| Resolve disagreements | ✅ | ✅ | ❌ | ❌ |

**Assignment Logic:**
```python
def get_next_task(project, user):
    member = ProjectMember.objects.get(project=project, user=user)

    if member.member_type == MemberTypes.REVIEWER:
        # Check review queue first
        review_task = get_next_review_task(project, user)
        if review_task:
            return review_task

    # All roles can get regular annotation tasks
    return get_next_annotation_task(project, user)
```

### 7. Import/Export Status Enumeration

**Source:** `label_studio/data_import/models.py`, `label_studio/data_export/models.py`

| Status | Value | Meaning | Transitions |
|--------|-------|---------|-------------|
| **CREATED** | "created" | Job queued, not started | → IN_PROGRESS (worker picks up) |
| **IN_PROGRESS** | "in_progress" | Currently processing | → COMPLETED (success) OR FAILED (error) |
| **FAILED** | "failed" | Error occurred | Terminal state (can retry by creating new job) |
| **COMPLETED** | "completed" | Successfully finished | Terminal state |

**Lifecycle:**
```
User uploads CSV
  ↓
FileUpload.objects.create(status="created")
  ↓
Background worker picks up
  ↓
FileUpload.status = "in_progress"
  ↓
Process rows: parse CSV → create Tasks → create Predictions
  ↓
If error: status = "failed", save traceback
If success: status = "completed", update counters
```

**Monitoring:**
```sql
-- Stuck imports (>1 hour in progress)
SELECT * FROM project_import
WHERE status = 'in_progress'
  AND updated_at < NOW() - INTERVAL '1 hour';

-- Failure analysis
SELECT
    project_id,
    COUNT(*) as total_imports,
    SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failures,
    ROUND(100.0 * SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) / COUNT(*), 2) as failure_rate
FROM project_import
GROUP BY project_id
HAVING failure_rate > 10;
```

### 8. WebhookAction Enumeration

**Source:** `label_studio/webhooks/models.py:96-250`

| Action | Trigger Event | Payload | Project/Workspace Scope |
|--------|---------------|---------|-------------------------|
| **PROJECT_CREATED** | New project created | Full project JSON | Workspace-only |
| **PROJECT_UPDATED** | Project settings changed | Updated project JSON | Both |
| **PROJECT_DELETED** | Project soft-deleted | Project ID only | Workspace-only |
| **TASKS_CREATED** | Tasks imported | Array of task JSON | Project or Workspace |
| **TASKS_DELETED** | Tasks removed | Array of task IDs | Project or Workspace |
| **ANNOTATION_CREATED** | Single annotation submitted | Annotation + nested task | Project or Workspace |
| **ANNOTATIONS_CREATED** | Batch annotations submitted | Array of annotations + tasks | Project or Workspace |
| **ANNOTATION_UPDATED** | Annotation modified | Updated annotation + task | Project or Workspace |
| **ANNOTATIONS_DELETED** | Annotations removed | Array of annotation IDs | Project or Workspace |
| **LABEL_LINK_CREATED** | Label library link added | LabelLink + Label | Project or Workspace |
| **LABEL_LINK_UPDATED** | Label library link modified | Updated LabelLink + Label | Project or Workspace |
| **LABEL_LINK_DELETED** | Label library link removed | LabelLink ID only | Project or Workspace |

**Payload Structure:**
```json
{
  "action": "ANNOTATION_CREATED",
  "project": 123,
  "organization": 1,
  "workspace": 45,
  "created_at": "2024-01-15T10:30:00Z",
  "annotation": {
    "id": 789,
    "completed_by": 67,
    "result": [...],
    "was_cancelled": false,
    "ground_truth": false,
    "created_at": "2024-01-15T10:30:00Z"
  },
  "task": {
    "id": 456,
    "data": {"text": "Sample task"},
    "is_labeled": true,
    "overlap": 1
  }
}
```

**Configuration:**
```python
# Workspace-level webhook (all projects)
webhook = Webhook.objects.create(
    workspace=workspace,
    url="https://analytics.company.com/webhook",
    send_for_all_actions=False,  # Selective actions
    send_payload=True,
    headers={"Authorization": "Bearer secret"},
    is_active=True
)

# Enable specific actions
WebhookAction.objects.create(webhook=webhook, action="ANNOTATION_CREATED")
WebhookAction.objects.create(webhook=webhook, action="ANNOTATION_UPDATED")

# Project-specific webhook
project_webhook = Webhook.objects.create(
    workspace=workspace,
    project=project,  # Scoped to this project only
    url="https://project-tracker.com/webhook",
    send_for_all_actions=True  # All allowed actions
)
```

---

## 📖 REAL-WORLD SCENARIO EXAMPLES

### Scenario 1: Medical Image Classification with Quality Control

**Context:** Hospital needs 10,000 X-ray images classified as Normal/Pneumonia/COVID with high accuracy.

**Requirements:**
- 2 radiologists annotate each image (overlap)
- 20% random sample reviewed by senior radiologist
- 5% golden reference images for quality benchmarking
- Track individual radiologist performance

**Implementation:**

```python
# 1. Create project
project = Project.objects.create(
    title="X-Ray Classification - Jan 2024",
    workspace=radiology_workspace,
    project_type="LABELLING",
    description="Classify chest X-rays for pathology detection",

    # Label configuration
    label_config="""
    <View>
      <Image name="xray" value="$image_url" zoom="true" zoomControl="true"/>
      <Choices name="diagnosis" toName="xray" choice="single" required="true">
        <Choice value="Normal"/>
        <Choice value="Pneumonia"/>
        <Choice value="COVID-19"/>
        <Choice value="Other"/>
      </Choices>
      <Choices name="confidence" toName="xray" choice="single" required="true">
        <Choice value="High"/>
        <Choice value="Medium"/>
        <Choice value="Low"/>
      </Choices>
      <TextArea name="notes" toName="xray" placeholder="Clinical notes (optional)"/>
    </View>
    """,

    # Quality control
    maximum_annotations=2,  # 2 radiologists per image
    overlap_cohort_percentage=100,  # All images get 2 annotations
    review_percentage=20,  # 20% to senior review
    golden_ref_percentage=5,  # 5% are validated golden references

    # Workflow
    sampling="UNIFORM",  # Random to avoid bias
    skip_queue="REQUEUE_FOR_OTHERS",  # Skips go to other radiologists
    show_annotation_history=False,  # Blind annotation

    # UI
    expert_instruction="""
    <h2>Classification Guidelines</h2>
    <ul>
      <li><b>Normal:</b> No visible pathology</li>
      <li><b>Pneumonia:</b> Consolidation, air bronchograms</li>
      <li><b>COVID-19:</b> Ground-glass opacities, bilateral distribution</li>
      <li><b>Other:</b> Other pathologies (specify in notes)</li>
    </ul>
    <p>When in doubt, mark confidence as Low and add clinical notes.</p>
    """,
    show_instruction=True,

    # State
    state="LIVE",
    is_published=True,
    created_by=admin_user
)

# 2. Add team members
radiologists = [dr_smith, dr_jones, dr_williams]
for radiologist in radiologists:
    ProjectMember.objects.create(
        project=project,
        user=radiologist,
        member_type=MemberTypes.ANNOTATOR,
        task_limit=0  # Unlimited
    )

# Senior reviewer
ProjectMember.objects.create(
    project=project,
    user=dr_senior,
    member_type=MemberTypes.REVIEWER,
    task_limit=0
)

# 3. Configure task distribution
ProjectOpsManager.objects.create(
    project=project,
    split_strategy="EQUAL",  # Divide 10,000 images equally
    max_limit=0
)

# 4. Import tasks with DICOM images
FileUpload.objects.create(
    project=project,
    user=admin_user,
    file=csv_file  # Contains: image_url, patient_id, study_date
)

# CSV format:
# image_url,patient_id,study_date
# s3://xrays/img001.dcm,PT12345,2024-01-10
# s3://xrays/img002.dcm,PT12346,2024-01-10

# 5. Create golden reference tasks
golden_tasks = Task.objects.filter(project=project).order_by('?')[:500]  # 5% of 10,000
for task in golden_tasks:
    Annotation.objects.create(
        task=task,
        completed_by=gold_standard_expert,
        ground_truth=True,
        result=[{
            "value": {"choices": ["Normal"]},  # Expert-validated label
            "from_name": "diagnosis",
            "to_name": "xray",
            "type": "choices"
        }]
    )

# 6. Setup webhook for progress tracking
webhook = Webhook.objects.create(
    workspace=radiology_workspace,
    project=project,
    url="https://hospital-analytics.com/labeling-progress",
    send_payload=True,
    headers={"Authorization": "Bearer hospital_token"},
    is_active=True
)
WebhookAction.objects.create(webhook=webhook, action="ANNOTATION_CREATED")

# 7. Create review project (nested)
review_project = Project.objects.create(
    title="X-Ray Classification - Review Stage",
    workspace=radiology_workspace,
    project_type="ASSESSMENT",
    label_config=project.label_config,  # Same interface
    show_annotation_history=True,  # Reviewers see original annotations
    is_published=True
)

ProjectNestedRelations.objects.create(
    parent_project=project,
    child_project=review_project,
    nesting_rules={
        "trigger": "on_completion",
        "sample_percentage": 20,  # 20% to review
        "copy_annotations": True  # Reviewers see originals
    },
    is_active=True
)
```

**Workflow Execution:**

1. **Dr. Smith logs in** → Gets random X-ray via `get_next_task()`
2. **Dr. Smith annotates** → Creates Annotation(diagnosis="Pneumonia", confidence="High")
3. **System checks**: Task has 1 annotation, maximum_annotations=2 → is_labeled=False
4. **Dr. Jones gets same task** → Overlap requirement
5. **Dr. Jones annotates** → Creates Annotation(diagnosis="Pneumonia", confidence="Medium")
6. **System checks**: Task has 2 annotations, maximum_annotations=2 → is_labeled=True
7. **System checks review_percentage=20** → Random 20% chance → Task goes to review queue
8. **Dr. Senior (reviewer)** → Sees task in review queue with both annotations
9. **Dr. Senior accepts** → Task marked as reviewed
10. **Webhook fires** → Hospital analytics receives progress update

**Quality Monitoring:**

```sql
-- Radiologist performance vs golden references
SELECT
    u.email as radiologist,
    COUNT(a.id) as total_annotations,
    SUM(CASE WHEN a.result = gt.result THEN 1 ELSE 0 END) as matches,
    ROUND(100.0 * SUM(CASE WHEN a.result = gt.result THEN 1 ELSE 0 END) / COUNT(a.id), 2) as accuracy_pct
FROM task_completion a
JOIN htx_user u ON a.completed_by_id = u.id
JOIN task t ON a.task_id = t.id
JOIN task_completion gt ON gt.task_id = t.id AND gt.ground_truth = TRUE
WHERE t.project_id = 123
  AND a.was_cancelled = FALSE
  AND a.ground_truth = FALSE
GROUP BY u.email
ORDER BY accuracy_pct DESC;
```

### Scenario 2: Multi-Language Sentiment Analysis with Active Learning

**Context:** E-commerce company needs sentiment analysis for customer reviews in 10 languages, wants to minimize annotation cost using ML.

**Requirements:**
- ML model pre-annotates high-confidence reviews
- Humans annotate low-confidence reviews (active learning)
- Different annotators per language
- Track ML model improvements over time

**Implementation:**

```python
# 1. Create project
project = Project.objects.create(
    title="Customer Review Sentiment - Active Learning",
    workspace=ecommerce_workspace,

    label_config="""
    <View>
      <Header value="Review Text"/>
      <Text name="review" value="$text"/>
      <Header value="Language: $language"/>

      <Choices name="sentiment" toName="review" choice="single" required="true">
        <Choice value="Very Negative"/>
        <Choice value="Negative"/>
        <Choice value="Neutral"/>
        <Choice value="Positive"/>
        <Choice value="Very Positive"/>
      </Choices>

      <Rating name="confidence" toName="review" maxRating="5" required="true"/>
    </View>
    """,

    # Active learning
    sampling="UNCERTAINTY",  # Prioritize low-confidence predictions
    model_version="sentiment-multilang-v1.0",
    show_collab_predictions=True,  # Show ML predictions to annotators
    min_annotations_to_start_training=1000,  # Retrain after 1000 new annotations

    # Quality
    maximum_annotations=1,  # Single annotator (cost optimization)
    overlap_cohort_percentage=10,  # Only 10% get double-annotation for validation

    # Workflow
    skip_queue="REQUEUE_FOR_OTHERS",
    show_instruction=True,
    expert_instruction="""
    <h3>Sentiment Guidelines</h3>
    <p>The ML model has pre-labeled this review. If you agree, confirm. If not, correct.</p>
    <ul>
      <li><b>Very Negative:</b> Extremely dissatisfied, demands refund</li>
      <li><b>Negative:</b> Unhappy, multiple complaints</li>
      <li><b>Neutral:</b> Mixed or factual statement</li>
      <li><b>Positive:</b> Satisfied, recommends product</li>
      <li><b>Very Positive:</b> Extremely happy, glowing review</li>
    </ul>
    """
)

# 2. Add language-specific annotators
languages = ["en", "es", "fr", "de", "it", "pt", "ja", "zh", "ar", "hi"]
for lang in languages:
    annotator = User.objects.get(email=f"annotator-{lang}@company.com")
    ProjectMember.objects.create(
        project=project,
        user=annotator,
        member_type=MemberTypes.ANNOTATOR
    )

# 3. Import reviews and ML predictions
import_job = ProjectImport.objects.create(
    project=project,
    preannotated_from_fields=["ml_prediction"],  # Use ML output
    commit_to_project=True,
    status="created"
)

# CSV format:
# text,language,ml_prediction,ml_confidence
# "Great product!",en,"{""sentiment"": ""Positive""}",0.92
# "Terrible quality",en,"{""sentiment"": ""Negative""}",0.87
# "Bueno pero caro",es,"{""sentiment"": ""Neutral""}",0.61

# Process import (background worker)
for row in csv_rows:
    task = Task.objects.create(
        project=project,
        data={"text": row["text"], "language": row["language"]}
    )

    # Create prediction
    Prediction.objects.create(
        task=task,
        model_version="sentiment-multilang-v1.0",
        result=[{
            "value": {"choices": [row["ml_prediction"]["sentiment"]]},
            "from_name": "sentiment",
            "to_name": "review",
            "type": "choices"
        }],
        score=row["ml_confidence"]  # Used for uncertainty sampling
    )

# 4. Configure language-based task filtering
for lang in languages:
    # Create view for each language
    view = DataManagerView.objects.create(
        project=project,
        title=f"Reviews - {lang.upper()}",
        data={"columns": ["text", "language", "sentiment"]}
    )

    filter_group = FilterGroup.objects.create(conjunction="and")
    Filter.objects.create(
        group=filter_group,
        column="language",
        operator="equal",
        value={"value": lang}
    )
    view.filter_group = filter_group
    view.save()

# 5. Setup ML retraining webhook
webhook = Webhook.objects.create(
    workspace=ecommerce_workspace,
    project=project,
    url="https://ml-pipeline.company.com/trigger-training",
    send_for_all_actions=False,
    headers={"X-API-Key": "ml_secret"},
    is_active=True
)
WebhookAction.objects.create(webhook=webhook, action="ANNOTATIONS_CREATED")
```

**Active Learning Workflow:**

1. **ML model predicts** → 100,000 reviews with confidence scores
2. **High confidence (>0.9)** → 70,000 reviews auto-labeled (not sent to humans)
3. **Low confidence (<0.9)** → 30,000 reviews sent to project
4. **Uncertainty sampling** → Lowest confidence reviews assigned first
5. **Annotator corrects** → Review with confidence=0.61 gets human annotation
6. **After 1000 annotations** → Webhook triggers ML retraining
7. **New model (v1.1)** → Re-predicts all unlabeled tasks
8. **Cycle repeats** → Model improves, fewer human annotations needed

**Cost Savings Analysis:**

```sql
-- Calculate annotation cost savings from ML pre-annotations
WITH predictions_stats AS (
    SELECT
        COUNT(*) as total_predictions,
        SUM(CASE WHEN score > 0.9 THEN 1 ELSE 0 END) as high_confidence,
        SUM(CASE WHEN score <= 0.9 THEN 1 ELSE 0 END) as low_confidence
    FROM task_prediction tp
    JOIN task t ON tp.task_id = t.id
    WHERE t.project_id = 456
      AND tp.model_version LIKE 'sentiment-multilang%'
),
annotation_stats AS (
    SELECT COUNT(*) as human_annotations
    FROM task_completion
    WHERE project_id = 456
      AND was_cancelled = FALSE
)
SELECT
    p.total_predictions,
    p.high_confidence as ml_auto_labeled,
    a.human_annotations,
    ROUND(100.0 * p.high_confidence / p.total_predictions, 2) as ml_coverage_pct,
    (p.total_predictions - a.human_annotations) * 0.50 as cost_saved_usd  -- $0.50 per annotation
FROM predictions_stats p, annotation_stats a;
```

### Scenario 3: Video Frame Annotation with Nested Review Pipeline

**Context:** Autonomous vehicle company needs bounding boxes on 1M video frames with strict quality requirements.

**Requirements:**
- 3-stage pipeline: Annotation → QA Review → Expert Adjudication
- Sequential sampling (annotate frames in order)
- Track which frames have disagreements
- Export to COCO format for model training

**Implementation:**

```python
# 1. Main annotation project
annotation_project = Project.objects.create(
    title="AV Dataset - Frame Annotation",
    workspace=av_workspace,

    label_config="""
    <View>
      <Image name="frame" value="$image_url" zoom="true"/>
      <RectangleLabels name="objects" toName="frame">
        <Label value="Car" background="red"/>
        <Label value="Pedestrian" background="blue"/>
        <Label value="Cyclist" background="green"/>
        <Label value="Traffic Sign" background="yellow"/>
        <Label value="Traffic Light" background="orange"/>
      </RectangleLabels>
    </View>
    """,

    # Sequential annotation (frame order matters)
    sampling="SEQUENCE",
    show_overlap_first=True,  # Prioritize frames needing multiple annotations

    # Quality control
    maximum_annotations=2,  # 2 annotators per frame
    overlap_cohort_percentage=100,

    # State
    state="LIVE",
    is_published=True
)

# 2. QA Review project (stage 2)
review_project = Project.objects.create(
    title="AV Dataset - QA Review",
    workspace=av_workspace,
    project_type="ASSESSMENT",
    label_config=annotation_project.label_config,
    show_annotation_history=True,  # Reviewers see original annotations
    sampling="SEQUENCE",
    maximum_annotations=1,  # Reviewer decision
    state="LIVE",
    is_published=True
)

ProjectNestedRelations.objects.create(
    parent_project=annotation_project,
    child_project=review_project,
    nesting_rules={
        "trigger": "on_completion",
        "sample_percentage": 100,  # All frames go to review
        "filter": {
            "min_annotations": 2,
            "max_agreement": 0.8  # Only frames with <80% agreement
        },
        "copy_annotations": True
    },
    is_active=True
)

# 3. Expert adjudication project (stage 3)
adjudication_project = Project.objects.create(
    title="AV Dataset - Expert Adjudication",
    workspace=av_workspace,
    project_type="ASSESSMENT",
    label_config=annotation_project.label_config,
    show_annotation_history=True,
    sampling="SEQUENCE",
    maximum_annotations=1,  # Expert makes final decision
    state="LIVE",
    is_published=True
)

ProjectNestedRelations.objects.create(
    parent_project=review_project,
    child_project=adjudication_project,
    nesting_rules={
        "trigger": "on_review_rejection",  # Only rejected frames
        "copy_annotations": True
    },
    is_active=True
)

# 4. Add team members
# Stage 1: Annotators
for annotator in junior_annotators:
    ProjectMember.objects.create(
        project=annotation_project,
        user=annotator,
        member_type=MemberTypes.ANNOTATOR,
        task_limit=5000  # 5000 frames per annotator
    )

# Stage 2: QA Reviewers
for reviewer in qa_reviewers:
    ProjectMember.objects.create(
        project=review_project,
        user=reviewer,
        member_type=MemberTypes.REVIEWER
    )

# Stage 3: Expert Adjudicators
for expert in senior_experts:
    ProjectMember.objects.create(
        project=adjudication_project,
        user=expert,
        member_type=MemberTypes.ADJUDICATOR
    )

# 5. Import video frames
for video_id in range(1, 1001):  # 1000 videos
    for frame_num in range(1, 1001):  # 1000 frames each = 1M total
        Task.objects.create(
            project=annotation_project,
            data={
                "image_url": f"s3://av-frames/video_{video_id}/frame_{frame_num:04d}.jpg",
                "video_id": video_id,
                "frame_number": frame_num,
                "timestamp": frame_num / 30.0  # 30 FPS
            }
        )

# 6. Setup export webhook
webhook = Webhook.objects.create(
    workspace=av_workspace,
    url="https://ml-training.av-company.com/new-annotations",
    send_for_all_actions=False
)
WebhookAction.objects.create(webhook=webhook, action="ANNOTATIONS_CREATED")
```

**Pipeline Workflow:**

```
Video Frame 001
  ↓
[STAGE 1: Annotation Project]
  → Annotator A: Draws bounding boxes
  → Annotator B: Draws bounding boxes
  → System calculates agreement: 75% (below 80% threshold)
  ↓
[STAGE 2: QA Review Project]
  → Frame appears in review queue (low agreement)
  → QA Reviewer: "Annotator A correct, B missed pedestrian"
  → Decision: REJECT
  ↓
[STAGE 3: Expert Adjudication Project]
  → Frame appears in adjudication queue (rejected)
  → Expert reviews both annotations + QA feedback
  → Expert creates authoritative annotation
  → Marked as GOLDEN REFERENCE
  ↓
[EXPORT]
  → Webhook triggers
  → Frame exported to COCO format
  → ML training pipeline ingests
```

**Agreement Calculation:**

```python
# Custom agreement calculation for bounding boxes
def calculate_bbox_agreement(annotation1, annotation2):
    boxes1 = annotation1.result
    boxes2 = annotation2.result

    matched_boxes = 0
    total_boxes = len(boxes1) + len(boxes2)

    for box1 in boxes1:
        for box2 in boxes2:
            iou = calculate_iou(box1, box2)
            if iou > 0.5 and box1['value']['rectanglelabels'] == box2['value']['rectanglelabels']:
                matched_boxes += 1
                break

    agreement = (2 * matched_boxes) / total_boxes if total_boxes > 0 else 0
    return agreement

# Store in control_weights
annotation_project.control_weights = {
    "objects": {
        "type": "RectangleLabels",
        "labels": {
            "Car": 1.0,
            "Pedestrian": 1.5,  # More important
            "Cyclist": 1.5,
            "Traffic Sign": 0.8,
            "Traffic Light": 0.8
        },
        "overall": 1.0
    }
}
annotation_project.save()
```

**Progress Monitoring:**

```sql
-- Pipeline stage funnel
SELECT
    'Stage 1: Annotation' as stage,
    COUNT(DISTINCT t.id) as tasks,
    COUNT(a.id) as annotations,
    SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) as completed
FROM task t
LEFT JOIN task_completion a ON a.task_id = t.id
WHERE t.project_id = 101
UNION ALL
SELECT
    'Stage 2: QA Review' as stage,
    COUNT(DISTINCT t.id) as tasks,
    COUNT(a.id) as annotations,
    SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) as completed
FROM task t
LEFT JOIN task_completion a ON a.task_id = t.id
WHERE t.project_id = 102
UNION ALL
SELECT
    'Stage 3: Adjudication' as stage,
    COUNT(DISTINCT t.id) as tasks,
    COUNT(a.id) as annotations,
    SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) as completed
FROM task t
LEFT JOIN task_completion a ON a.task_id = t.id
WHERE t.project_id = 103;
```

---

## ❓ FAQ WITH SQL QUERY EXAMPLES

### Q1: How do I find projects with stuck imports (>1 hour)?

```sql
SELECT
    pi.id as import_id,
    p.id as project_id,
    p.title as project_name,
    pi.status,
    pi.created_at,
    pi.updated_at,
    EXTRACT(EPOCH FROM (NOW() - pi.updated_at))/3600 as hours_stuck
FROM project_import pi
JOIN project p ON pi.project_id = p.id
WHERE pi.status = 'in_progress'
  AND pi.updated_at < NOW() - INTERVAL '1 hour'
ORDER BY hours_stuck DESC;
```

### Q2: How do I find all tasks assigned to a specific annotator?

```sql
-- Tasks already annotated by user
SELECT t.*
FROM task t
JOIN task_completion a ON a.task_id = t.id
WHERE a.completed_by_id = 42
  AND a.was_cancelled = FALSE;

-- Tasks NOT yet annotated by user (available in queue)
SELECT t.*
FROM task t
WHERE t.project_id = 123
  AND t.is_labeled = FALSE
  AND NOT EXISTS (
    SELECT 1 FROM task_completion a
    WHERE a.task_id = t.id
      AND a.completed_by_id = 42
  );
```

### Q3: How do I calculate inter-annotator agreement for a project?

```sql
-- Pair-wise agreement for tasks with 2+ annotations
WITH task_pairs AS (
    SELECT
        a1.task_id,
        a1.id as annotation1_id,
        a2.id as annotation2_id,
        a1.completed_by_id as annotator1,
        a2.completed_by_id as annotator2,
        a1.result as result1,
        a2.result as result2
    FROM task_completion a1
    JOIN task_completion a2 ON a1.task_id = a2.task_id AND a1.id < a2.id
    WHERE a1.project_id = 123
      AND a1.was_cancelled = FALSE
      AND a2.was_cancelled = FALSE
)
SELECT
    annotator1,
    annotator2,
    COUNT(*) as tasks_in_common,
    SUM(CASE WHEN result1 = result2 THEN 1 ELSE 0 END) as agreements,
    ROUND(100.0 * SUM(CASE WHEN result1 = result2 THEN 1 ELSE 0 END) / COUNT(*), 2) as agreement_pct
FROM task_pairs
GROUP BY annotator1, annotator2
ORDER BY agreement_pct DESC;
```

### Q4: How do I find projects with low completion rates?

```sql
SELECT
    p.id,
    p.title,
    p.state,
    COUNT(t.id) as total_tasks,
    SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) as completed_tasks,
    ROUND(100.0 * SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) / NULLIF(COUNT(t.id), 0), 2) as completion_pct,
    p.created_at,
    EXTRACT(DAY FROM NOW() - p.created_at) as days_since_creation
FROM project p
LEFT JOIN task t ON t.project_id = p.id
WHERE p.workspace_id = 45
  AND p.is_deleted = FALSE
  AND p.state = 'live'
GROUP BY p.id
HAVING
    COUNT(t.id) > 0
    AND ROUND(100.0 * SUM(CASE WHEN t.is_labeled THEN 1 ELSE 0 END) / COUNT(t.id), 2) < 50
ORDER BY completion_pct ASC;
```

### Q5: How do I find annotators with productivity below average?

```sql
WITH annotator_stats AS (
    SELECT
        pm.user_id,
        u.email,
        COUNT(a.id) as annotations_count,
        COUNT(a.id) / NULLIF(EXTRACT(DAY FROM NOW() - pm.created_at), 0) as annotations_per_day
    FROM project_member pm
    JOIN htx_user u ON pm.user_id = u.id
    JOIN task_completion a ON a.completed_by_id = pm.user_id
    WHERE pm.project_id = 123
      AND pm.member_type = 4  -- ANNOTATOR
      AND a.was_cancelled = FALSE
    GROUP BY pm.user_id, u.email, pm.created_at
),
avg_productivity AS (
    SELECT AVG(annotations_per_day) as avg_rate
    FROM annotator_stats
)
SELECT
    s.user_id,
    s.email,
    s.annotations_count,
    ROUND(s.annotations_per_day::numeric, 2) as annotations_per_day,
    ROUND(ap.avg_rate::numeric, 2) as team_average,
    ROUND(100.0 * (s.annotations_per_day - ap.avg_rate) / NULLIF(ap.avg_rate, 0), 2) as variance_pct
FROM annotator_stats s, avg_productivity ap
WHERE s.annotations_per_day < ap.avg_rate
ORDER BY variance_pct ASC;
```

### Q6: How do I find tasks with disagreements (low inter-annotator agreement)?

```sql
-- Assuming choice-based annotations
WITH task_annotations AS (
    SELECT
        t.id as task_id,
        t.data,
        COUNT(a.id) as annotation_count,
        COUNT(DISTINCT a.result::jsonb->0->'value'->'choices'->0) as unique_choices
    FROM task t
    JOIN task_completion a ON a.task_id = t.id
    WHERE t.project_id = 123
      AND a.was_cancelled = FALSE
      AND t.is_labeled = TRUE
    GROUP BY t.id
)
SELECT
    task_id,
    data,
    annotation_count,
    unique_choices,
    CASE
        WHEN unique_choices = 1 THEN 'Perfect Agreement'
        WHEN unique_choices = annotation_count THEN 'Complete Disagreement'
        ELSE 'Partial Disagreement'
    END as agreement_status
FROM task_annotations
WHERE annotation_count >= 2
  AND unique_choices > 1  -- Disagreement
ORDER BY unique_choices DESC, annotation_count DESC;
```

### Q7: How do I audit webhook delivery failures?

```sql
-- Note: Webhook logs typically stored in separate webhook_log table
-- This example assumes webhook_log exists

SELECT
    wl.id,
    w.url,
    wa.action,
    wl.status_code,
    wl.response_body,
    wl.created_at,
    wl.retries
FROM webhook_log wl
JOIN webhook w ON wl.webhook_id = w.id
JOIN webhook_action wa ON wa.webhook_id = w.id
WHERE w.project_id = 123
  AND wl.status_code >= 400  -- HTTP errors
  AND wl.created_at > NOW() - INTERVAL '24 hours'
ORDER BY wl.created_at DESC;
```

### Q8: How do I find which label configurations are most common?

```sql
-- Group projects by label_config hash
SELECT
    label_config_hash,
    COUNT(*) as project_count,
    ARRAY_AGG(title ORDER BY created_at DESC) as project_titles,
    MAX(created_at) as most_recent_use
FROM project
WHERE workspace_id = 45
  AND is_deleted = FALSE
GROUP BY label_config_hash
HAVING COUNT(*) > 1  -- Used by multiple projects
ORDER BY project_count DESC
LIMIT 10;
```

### Q9: How do I export all annotations for a project in JSON format?

```sql
SELECT json_build_object(
    'task_id', t.id,
    'task_data', t.data,
    'annotations', json_agg(
        json_build_object(
            'annotation_id', a.id,
            'completed_by', u.email,
            'result', a.result,
            'created_at', a.created_at,
            'was_cancelled', a.was_cancelled,
            'ground_truth', a.ground_truth
        ) ORDER BY a.created_at
    )
) as export_data
FROM task t
LEFT JOIN task_completion a ON a.task_id = t.id
LEFT JOIN htx_user u ON a.completed_by_id = u.id
WHERE t.project_id = 123
GROUP BY t.id
ORDER BY t.id;
```

### Q10: How do I find orphaned tasks (no project or deleted project)?

```sql
-- Tasks with NULL project_id or pointing to deleted projects
SELECT
    t.id as task_id,
    t.project_id,
    t.data,
    t.created_at,
    p.is_deleted as project_deleted,
    p.title as project_title
FROM task t
LEFT JOIN project p ON t.project_id = p.id
WHERE t.project_id IS NULL
   OR p.is_deleted = TRUE
ORDER BY t.created_at DESC;
```

---

## ⚠️ COMMON PITFALLS & GOTCHAS

### Pitfall 1: Changing label_config Breaks Existing Annotations

**WRONG:**
```python
project.label_config = """
<View>
    <Text name="review" value="$text"/>
    <Choices name="NEW_sentiment" toName="review">  <!-- Changed control name -->
        <Choice value="positive"/>
    </Choices>
</View>
"""
project.save()  # DANGER: Orphans existing annotations!
```

**Problem:**
Existing annotations reference `from_name="sentiment"`. New config uses `"NEW_sentiment"`. Annotations become invalid.

**RIGHT:**
```python
# Validate before changing
project.validate_config(new_config_string, strict=True)  # Throws error if incompatible

# OR use migration approach
if project.annotations.exists():
    # Create new project with new config
    new_project = Project.objects.create(...)
    # Migrate tasks
    tasks = Task.objects.filter(project=old_project)
    tasks.update(project=new_project)
else:
    # No annotations yet, safe to change
    project.label_config = new_config_string
    project.save()
```

**Best Practice:**
- Never change control names (`name="..."`) after annotations exist
- Add new controls instead of modifying existing ones
- Use `validate_config()` before saving

---

### Pitfall 2: maximum_annotations vs overlap Confusion

**WRONG:**
```python
project.maximum_annotations = 3  # Want 3 annotations per task
project.overlap_cohort_percentage = 50  # But only 50% get overlap!
```

**Problem:**
Only 50% of tasks will get 3 annotations. Other 50% get 1 annotation.

**RIGHT:**
```python
project.maximum_annotations = 3
project.overlap_cohort_percentage = 100  # ALL tasks get 3 annotations

# OR intentional stratified sampling
project.maximum_annotations = 3
project.overlap_cohort_percentage = 20  # 20% get 3 annotations, 80% get 1
```

**Explanation:**
- `maximum_annotations`: How many annotations when overlap is applied
- `overlap_cohort_percentage`: What % of tasks get that overlap
- If overlap_cohort_percentage < 100, some tasks only get 1 annotation

---

### Pitfall 3: skip_queue Doesn't Prevent Re-assignment

**WRONG Assumption:**
```python
project.skip_queue = "REQUEUE_FOR_OTHERS"
# Annotator skips task
# Assumption: Task will NEVER go back to same annotator
```

**Problem:**
If all other annotators also skip, task may circle back to original annotator.

**RIGHT Understanding:**
```python
# skip_queue="REQUEUE_FOR_OTHERS" only excludes for THIS ROUND
# Once all annotators skip, task pool resets

# To permanently exclude:
def get_next_task_custom(project, user):
    skipped_tasks = Annotation.objects.filter(
        completed_by=user,
        was_cancelled=True,
        project=project
    ).values_list('task_id', flat=True)

    return Task.objects.filter(
        project=project,
        is_labeled=False
    ).exclude(
        id__in=skipped_tasks  # Permanently exclude
    ).first()
```

---

### Pitfall 4: ProjectSummary Not Updated in Real-Time

**WRONG:**
```python
# Import 1000 tasks
for row in csv_rows:
    Task.objects.create(project=project, data=row)

# Immediately check summary
summary = project.summary.all_data_columns  # STALE DATA!
```

**Problem:**
ProjectSummary updated via signals, which may have delay in high-load scenarios.

**RIGHT:**
```python
# Option 1: Force refresh
project.summary.update_data_columns(newly_created_tasks)
summary = project.summary.all_data_columns

# Option 2: Query directly (slower but accurate)
from django.db.models import JSONField
data_columns = Task.objects.filter(project=project).values_list('data__keys()', flat=True).distinct()
```

---

### Pitfall 5: Webhook Payload Size Limits

**WRONG:**
```python
webhook = Webhook.objects.create(
    url="https://myserver.com/webhook",
    send_payload=True,
    send_for_all_actions=True
)

# Import 10,000 tasks at once
# Webhook tries to send TASKS_CREATED with 10,000 task JSONs
# Payload: 50MB+ → HTTP 413 Entity Too Large
```

**Problem:**
Large batch operations create massive webhook payloads that fail.

**RIGHT:**
```python
# Option 1: Disable payload for bulk actions
webhook.send_payload = False  # Only send action type, not data

# Option 2: Use selective actions
webhook.send_for_all_actions = False
WebhookAction.objects.create(webhook=webhook, action="ANNOTATION_CREATED")  # Not TASKS_CREATED

# Option 3: Paginate imports
for chunk in chunked(tasks, 100):  # Import 100 at a time
    bulk_create_tasks(chunk)
```

---

### Pitfall 6: Nested Projects Create Circular Dependencies

**WRONG:**
```python
# Project A → Project B
ProjectNestedRelations.objects.create(
    parent_project=project_a,
    child_project=project_b
)

# Project B → Project A (CIRCULAR!)
ProjectNestedRelations.objects.create(
    parent_project=project_b,
    child_project=project_a
)
```

**Problem:**
Tasks cycle infinitely between projects, never completing.

**RIGHT:**
```python
# Enforce directed acyclic graph (DAG)
def create_nesting(parent, child):
    # Check for cycles
    def has_cycle(project, target, visited=set()):
        if project == target:
            return True
        if project in visited:
            return False
        visited.add(project)

        for relation in ProjectNestedRelations.objects.filter(parent_project=project):
            if has_cycle(relation.child_project, target, visited):
                return True
        return False

    if has_cycle(child, parent):
        raise ValidationError("Circular nesting detected!")

    ProjectNestedRelations.objects.create(parent_project=parent, child_project=child)
```

---

### Pitfall 7: Deleted Projects Leave Orphaned Data

**WRONG:**
```python
project.delete()  # CASCADE deletes tasks, annotations, etc.
# BUT: FileUpload, ProjectImport, webhooks in logs still reference project_id!
```

**Problem:**
Foreign keys with CASCADE delete remove core data, but related logs/audits broken.

**RIGHT:**
```python
# Soft delete
project.is_deleted = True
project.state = "COMPLETED"
project.save()

# OR explicit cleanup
def safe_delete_project(project):
    # Archive exports first
    for export in Export.objects.filter(project=project):
        archive_export(export)

    # Cancel pending imports
    ProjectImport.objects.filter(project=project, status__in=["created", "in_progress"]).update(status="cancelled")

    # Deactivate webhooks
    Webhook.objects.filter(project=project).update(is_active=False)

    # Now safe to delete
    project.delete()
```

---

### Pitfall 8: DataManagerView Filters Don't Update Automatically

**WRONG:**
```python
view = DataManagerView.objects.create(
    project=project,
    filter_group=filter_group  # Filters for status="pending"
)

# 100 tasks completed
# View still shows old "pending" tasks in cache
```

**Problem:**
Views cache prepared params. Filters apply to stale data.

**RIGHT:**
```python
# Explicitly refresh view
view = DataManagerView.objects.get(id=view_id)
params = view.get_prepare_tasks_params()  # Recomputes filters

# OR set cache timeout
# In view serializer:
def get_tasks(self):
    cache_key = f"view_{self.id}_tasks"
    tasks = cache.get(cache_key)
    if not tasks:
        tasks = self.apply_filters()
        cache.set(cache_key, tasks, timeout=300)  # 5 min cache
    return tasks
```

---

### Pitfall 9: Golden Reference Tasks Counted Toward Completion

**WRONG:**
```python
# Create golden reference
Annotation.objects.create(
    task=task,
    ground_truth=True,
    completed_by=expert,
    result=[...]
)

# Check completion
task.refresh_from_db()
task.is_labeled  # TRUE! Even though no real annotators worked on it
```

**Problem:**
Ground truth annotations count toward `maximum_annotations`, marking task complete prematurely.

**RIGHT:**
```python
# Exclude ground truth from completion logic
def update_task_labeled_status(task):
    real_annotations = task.annotations.filter(
        was_cancelled=False,
        ground_truth=False  # Exclude ground truth
    ).count()

    task.is_labeled = (real_annotations >= task.project.maximum_annotations)
    task.save()
```

---

### Pitfall 10: Task Locks Not Released on Browser Crash

**WRONG Assumption:**
```python
# User opens task → task locked
# Browser crashes → lock remains FOREVER
# No other annotator can access task
```

**Problem:**
Task locks (collaborative annotation feature) use timeouts, but long timeouts cause blockage.

**RIGHT:**
```python
# Configure reasonable lock timeout
TASK_LOCK_TTL = 600  # 10 minutes

# Periodic cleanup job
def release_stale_locks():
    from django.core.cache import cache
    # Locks stored in cache with TTL
    # Auto-expire after TASK_LOCK_TTL

# OR manual unlock
def force_unlock_task(task_id):
    cache_key = f"task_lock_{task_id}"
    cache.delete(cache_key)
```

---

## 🔀 STATE TRANSITION DIAGRAMS

### Project Lifecycle States

```
                    ┌──────────────────┐
                    │   Created        │
                    │   state=PENDING  │
                    └────────┬─────────┘
                             │
                             │ Admin starts configuration
                             ▼
                    ┌──────────────────┐
                    │  Configuration   │
        ┌──────────▶│  state=BUILDING_ │
        │           │    WORKFLOW      │
        │           └────────┬─────────┘
        │                    │
        │                    │ Admin publishes (is_published=True)
        │                    ▼
        │           ┌──────────────────┐
        │           │   Production     │◀──────┐
        │           │   state=LIVE     │       │
        │           └────┬──────────┬──┘       │
        │                │          │          │
        │                │          │          │ Admin resumes
        │                │          │          │
        │                │          │ Admin pauses
        │                │          ▼          │
        │                │  ┌──────────────┐   │
        │                │  │   Paused     │   │
        │                │  │ state=PAUSED │───┘
        │                │  └──────────────┘
        │                │
        │                │ All tasks complete OR manual completion
        │                ▼
        │           ┌──────────────────┐
        │           │   Finished       │
        └───────────│ state=COMPLETED  │
         Admin      └──────────────────┘
         reopens
```

**State Transition Rules:**

| From State | To State | Trigger | Conditions |
|------------|----------|---------|------------|
| PENDING | BUILDING_WORKFLOW | Admin action | User starts configuration |
| BUILDING_WORKFLOW | LIVE | `is_published=True` | label_config set, tasks imported |
| LIVE | PAUSED | Admin action | Temporary halt needed |
| PAUSED | LIVE | Admin action | Resume work |
| LIVE | COMPLETED | Auto or manual | All tasks reach maximum_annotations |
| COMPLETED | LIVE | Admin action | Need more annotations |
| Any | PENDING | Admin action | Reset project (rare) |

---

### Task Lifecycle States

```
                     ┌──────────────┐
                     │   Imported   │
                     │ is_labeled=  │
                     │    FALSE     │
                     └──────┬───────┘
                            │
                            │ Task enters annotation queue
                            ▼
              ┌─────────────────────────┐
              │   Annotation Queue      │
              │   (Available to users)  │
              └──────┬──────────────────┘
                     │
                     │ User picks task
                     ▼
              ┌──────────────┐
              │   Locked     │  (Collaborative mode only)
              │ (Temporary)  │
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        │ Skip       │ Submit     │ Timeout
        │            │            │
        ▼            ▼            ▼
   ┌────────┐  ┌──────────┐  ┌────────┐
   │Skipped │  │Annotated │  │Released│
   │(depends│  │(partial) │  │        │
   │on skip │  └────┬─────┘  └────────┘
   │_queue) │       │             │
   └────────┘       │             │
        │           │             │
        │           ▼             │
        │   ┌──────────────┐     │
        │   │  Check if    │     │
        └──▶│  completed   │◀────┘
            │(annotations  │
            │ >= maximum)  │
            └──────┬───────┘
                   │
         ┌─────────┴──────────┐
         │                    │
         │ YES                │ NO
         ▼                    ▼
    ┌─────────┐        ┌────────────┐
    │Completed│        │Back to     │
    │is_labeled│        │Queue       │
    │= TRUE   │        │            │
    └────┬────┘        └──────┬─────┘
         │                    │
         │                    └──────────┐
         │                               │
         │ If review_percentage > 0      │
         ▼                               │
    ┌─────────┐                          │
    │ Review  │                          │
    │ Queue?  │                          │
    └────┬────┘                          │
         │                               │
    ┌────┴────┐                          │
    │ YES│ NO │                          │
    ▼    ▼    ▼                          ▼
┌────────┐ ┌────────┐           ┌────────────┐
│To      │ │Finished│           │ Needs More │
│Reviewer│ │        │           │ Annotations│
└────────┘ └────────┘           └────────────┘
```

**Task State Calculation Logic:**

```python
def update_task_is_labeled(task):
    project = task.project

    # Count non-cancelled, non-ground-truth annotations
    valid_annotations = task.annotations.filter(
        was_cancelled=False,
        ground_truth=False
    ).count()

    # Check if maximum reached
    if valid_annotations >= project.maximum_annotations:
        task.is_labeled = True
    else:
        task.is_labeled = False

    task.save()

    # If labeled and review required
    if task.is_labeled and project.review_percentage > 0:
        if random.randint(1, 100) <= project.review_percentage:
            send_to_review_queue(task)
```

---

### Annotation Workflow States

```
                  ┌────────────────┐
                  │  Draft Created │
                  │  (optional)    │
                  └────────┬───────┘
                           │
                           ▼
                  ┌────────────────┐
                  │   Submitted    │
                  │ was_cancelled= │
                  │     FALSE      │
                  └────────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
    ┌─────────────┐ ┌──────────┐ ┌──────────┐
    │  Ground     │ │  Regular │ │  Review  │
    │  Truth      │ │Annotation│ │  Queue   │
    │ground_truth │ │          │ │          │
    │  = TRUE     │ │          │ │          │
    └─────────────┘ └────┬─────┘ └────┬─────┘
                         │            │
                         │            ▼
                         │      ┌──────────┐
                         │      │ Reviewer │
                         │      │ Assesses │
                         │      └────┬─────┘
                         │           │
                         │    ┌──────┴──────┐
                         │    │             │
                         │    ▼             ▼
                         │ ┌───────┐   ┌────────┐
                         │ │Accepted│   │Rejected│
                         │ └───────┘   └────┬───┘
                         │                  │
                         │                  ▼
                         │            ┌──────────┐
                         │            │ To       │
                         │            │Adjudica- │
                         │            │tion?     │
                         │            └────┬─────┘
                         │                 │
                         ▼                 ▼
                  ┌────────────┐    ┌────────────┐
                  │  Final     │    │ Adjudicator│
                  │ Annotation │◀───│  Decision  │
                  └────────────┘    └────────────┘
```

---

## 🔗 RELATIONSHIP DIAGRAMS (ASCII)

### Core Project Relationships

```
┌───────────────────┐
│   Organization    │
└─────────┬─────────┘
          │ 1:N
          ▼
┌───────────────────┐
│    Workspace      │
└─────────┬─────────┘
          │ 1:N
          ▼
┌───────────────────┐
│     Project       │◀──────┐
└──┬────┬────┬─────┬┘       │
   │    │    │     │         │
   │1:N │1:N │1:N  │1:N      │N:M (nested)
   │    │    │     │         │
   ▼    ▼    ▼     ▼         │
┌────┐┌────┐┌────┐┌────────────────┐
│Task││Memb││View││ProjectNested   │
│    ││er  ││    ││Relations       │
└─┬──┘└────┘└────┘└────────────────┘
  │
  │1:N
  ▼
┌────────────┐
│ Annotation │
└────────────┘
```

### Data Import Flow

```
┌──────────┐      ┌───────────────┐      ┌─────────────┐
│   User   │─────▶│  FileUpload   │─────▶│ProjectImport│
└──────────┘      └───────┬───────┘      └──────┬──────┘
                          │                     │
                          │                     │ Creates
                          │ Parses              │
                          │                     ▼
                          │              ┌────────────┐
                          │              │   Task     │
                          │              └──────┬─────┘
                          │                     │
                          │                     │ Creates (if pre-annotated)
                          ▼                     ▼
                   ┌────────────┐        ┌────────────┐
                   │   Media    │        │ Prediction │
                   │ Generation │        │            │
                   │  (TTS/etc) │        └────────────┘
                   └────────────┘
```

### Webhook Event Flow

```
┌──────────────┐
│  Annotation  │
│   Created    │
└──────┬───────┘
       │
       │ Triggers signal
       ▼
┌──────────────────────┐
│ post_save signal     │
│ (annotation_created) │
└──────┬───────────────┘
       │
       │ Queries active webhooks
       ▼
┌──────────────────────┐
│  Webhook.objects     │
│  .filter(is_active)  │
│  .filter(actions     │
│   contains           │
│  "ANNOTATION_CREATED"│
│  )                   │
└──────┬───────────────┘
       │
       │ For each webhook
       ▼
┌──────────────────────┐
│  Serialize payload   │
│  (annotation + task) │
└──────┬───────────────┘
       │
       │ HTTP POST
       ▼
┌──────────────────────┐
│  External System     │
│  (webhook.url)       │
└──────────────────────┘
```

### Nested Project Review Pipeline

```
┌────────────────────┐
│ Annotation Project │
│   (Stage 1)        │
└─────────┬──────────┘
          │
          │ Task completed
          │ nesting_rules.trigger="on_completion"
          │ sample_percentage=20%
          ▼
┌────────────────────┐
│  Review Project    │
│   (Stage 2)        │
└─────────┬──────────┘
          │
          │ Review result="rejected"
          │ nesting_rules.trigger="on_review_rejection"
          ▼
┌────────────────────┐
│ Adjudication       │
│  Project           │
│  (Stage 3)         │
└────────────────────┘
```

---

## 🎓 ONBOARDING & LEARNING PATH

### For New Developers

**Week 1: Foundation**
1. Read this documentation top-to-bottom
2. Understand Project→Task→Annotation hierarchy
3. Set up local dev environment
4. Create simple project via Django shell:
   ```python
   from projects.models import Project
   p = Project.objects.create(title="Test", label_config="<View></View>")
   ```

**Week 2: Configuration**
1. Learn label_config XML syntax
2. Create projects with different UI controls (Choices, RectangleLabels, TextArea)
3. Understand parsed_label_config caching
4. Experiment with quality control settings (maximum_annotations, review_percentage)

**Week 3: Workflows**
1. Study sampling strategies (SEQUENCE, UNIFORM, UNCERTAINTY)
2. Trace `get_next_task()` logic in debugger
3. Create nested project pipeline
4. Configure webhooks and test payload delivery

**Week 4: Data Pipelines**
1. Import CSV files with various formats
2. Create predictions for active learning
3. Export annotations in JSON/CSV
4. Monitor ProjectImport status transitions

### For Annotators

**Day 1: Basics**
- What is a project?
- How to get assigned tasks
- How to submit annotations
- When to use skip button

**Day 2: Quality**
- Understanding instructions (expert_instruction)
- What are ground truth tasks?
- Why some tasks require multiple annotations
- How reviewer feedback works

**Day 3: Efficiency**
- Hotkeys for common actions
- Understanding task sampling (why you see certain tasks first)
- How to handle edge cases (empty annotations, unclear guidelines)

### For Project Managers

**Setup Phase:**
1. Clone template project or create from scratch
2. Configure label_config based on data type
3. Set quality control parameters (overlap, review %)
4. Import initial dataset
5. Create golden reference tasks
6. Add team members with roles

**Monitoring Phase:**
1. Check ProjectSummary for progress
2. Review annotation quality metrics
3. Identify slow/low-quality annotators
4. Adjust review_percentage based on results
5. Export data for model training

**Optimization Phase:**
1. Analyze bottlenecks (stuck tasks, skipped tasks)
2. Refine expert_instruction based on common errors
3. Adjust sampling strategy if needed
4. Scale team up/down based on velocity

---

## 📚 HISTORICAL CONTEXT & DESIGN DECISIONS

### Why XML for label_config?

**Historical Context:**
Label Studio initially used a custom JSON schema for UI definition. In v0.8 (2020), switched to XML to leverage existing LSF (Label Studio Frontend) parser and enable more complex hierarchical structures.

**Design Decision:**
- **Pros**: Hierarchical, extensible, supports nested tags
- **Cons**: Verbose, error-prone for manual editing
- **Alternative Considered**: YAML (rejected due to indentation brittleness)

**Current State:**
XML stored in `label_config` field, parsed to JSON in `parsed_label_config` for performance. Validation layer prevents breaking changes.

---

### Why JSONB for control_weights?

**Historical Context:**
Early versions (v0.x) calculated agreement metrics on-the-fly, causing severe performance issues on large projects (>10K tasks).

**Design Decision:**
v1.2 introduced `control_weights` as JSONB to pre-calculate and cache weights for each control tag.

**Structure:**
```json
{
  "sentiment": {
    "type": "Choices",
    "labels": {
      "positive": 1.0,
      "negative": 1.0,
      "neutral": 0.5
    },
    "overall": 1.0
  }
}
```

**Performance Impact:**
Agreement queries went from 30s+ to <1s on 100K annotation projects.

---

### Why overlap_cohort_percentage?

**Historical Context:**
Originally, all tasks got `maximum_annotations`. For large projects (1M+ tasks), this was cost-prohibitive.

**Design Decision:**
v1.5 added `overlap_cohort_percentage` to enable stratified sampling: annotate most tasks once, but get overlap on a subset for quality validation.

**Business Use Case:**
- 100K tasks, budget for 120K annotations
- Set maximum_annotations=3, overlap_cohort_percentage=10
- 90K tasks get 1 annotation each = 90K
- 10K tasks get 3 annotations each = 30K
- Total = 120K annotations

---

### Why ProjectSummary as Separate Table?

**Historical Context:**
Aggregation queries (`COUNT`, `SUM`) on Task/Annotation tables were killing database performance.

**Design Decision:**
v1.0 introduced `ProjectSummary` as denormalized cache, updated via Django signals.

**Trade-offs:**
- **Pro**: Instant access to counts without aggregation
- **Con**: Potential data inconsistency if signals fail
- **Mitigation**: Periodic reconciliation job verifies summary accuracy

---

### Why Soft Delete (is_deleted) Instead of Hard Delete?

**Historical Context:**
Early versions used hard deletes. Multiple production incidents where customers accidentally deleted projects with months of work.

**Design Decision:**
v0.9 added `is_deleted` flag for soft deletion with 30-day grace period.

**Implementation:**
```python
# Default manager excludes deleted
Project.objects.all()  # WHERE is_deleted=FALSE

# All objects manager includes deleted
Project.all_objects.all()  # No filter

# Recovery
project.is_deleted = False
project.save()
```

---

## 🛠️ TROUBLESHOOTING GUIDE

### Issue 1: Tasks Not Appearing in Queue

**Symptoms:**
Annotator logs in, clicks "Next Task", gets "No tasks available" message.

**Diagnostic Steps:**

1. **Check project state:**
   ```sql
   SELECT id, title, state, is_published FROM project WHERE id = 123;
   ```
   - state must be "live"
   - is_published must be TRUE

2. **Check if user is project member:**
   ```sql
   SELECT * FROM project_member WHERE project_id = 123 AND user_id = 42;
   ```
   - Must have record with valid member_type

3. **Check if tasks exist and unlabeled:**
   ```sql
   SELECT COUNT(*) FROM task WHERE project_id = 123 AND is_labeled = FALSE;
   ```
   - If 0, all tasks completed

4. **Check if user already annotated all tasks:**
   ```sql
   SELECT COUNT(*) FROM task t
   WHERE t.project_id = 123
     AND t.is_labeled = FALSE
     AND NOT EXISTS (
       SELECT 1 FROM task_completion a
       WHERE a.task_id = t.id AND a.completed_by_id = 42
     );
   ```
   - If 0, user exhausted their queue

5. **Check task limits:**
   ```python
   from projects.models import ProjectOpsManager
   ops_mgr = ProjectOpsManager.objects.get(project_id=123)
   limit = ops_mgr.get_limit(user)
   user_annotation_count = Annotation.objects.filter(project_id=123, completed_by=user).count()
   # If user_annotation_count >= limit, they hit cap
   ```

**Common Fixes:**
- Set `is_published=True`
- Add user to ProjectMember
- Increase task limit in ProjectOpsManager
- Import more tasks

---

### Issue 2: Import Stuck in "in_progress"

**Symptoms:**
FileUpload or ProjectImport shows status="in_progress" for hours.

**Diagnostic Steps:**

1. **Check background worker:**
   ```bash
   # Are Celery/RQ workers running?
   ps aux | grep celery
   ```

2. **Check import logs:**
   ```sql
   SELECT * FROM project_import WHERE id = 789;
   -- Look at error_traceback column
   ```

3. **Check file upload:**
   ```sql
   SELECT * FROM file_upload WHERE id = 456;
   -- Look at unsuccessful_rows, error details
   ```

**Common Causes:**
- Background worker crashed
- CSV parsing error (invalid JSON in column)
- Database connection timeout (huge files)
- S3 credentials invalid (can't access external files)

**Fixes:**
- Restart background workers
- Fix CSV formatting (escape quotes, valid JSON)
- Split large files into chunks
- Validate S3 credentials

---

### Issue 3: Annotations Not Counting Toward Completion

**Symptoms:**
Task has 2 annotations but is_labeled=FALSE despite maximum_annotations=2.

**Diagnostic Steps:**

1. **Check annotation flags:**
   ```sql
   SELECT id, was_cancelled, ground_truth FROM task_completion WHERE task_id = 456;
   ```
   - was_cancelled=TRUE doesn't count
   - ground_truth=TRUE might not count (depending on logic)

2. **Check project config:**
   ```sql
   SELECT maximum_annotations, overlap_cohort_percentage FROM project WHERE id = 123;
   ```
   - If overlap_cohort_percentage < 100, some tasks only need 1 annotation

3. **Check task overlap field:**
   ```sql
   SELECT id, overlap FROM task WHERE id = 456;
   ```
   - Task.overlap might be set to 1 even if project.maximum_annotations=2

**Common Fixes:**
- Ensure annotations have was_cancelled=FALSE
- Set overlap_cohort_percentage=100
- Run `project._update_tasks_states()` to recalculate
- Check if skip_queue="IGNORE_SKIPPED" is creating cancelled annotations

---

### Issue 4: Low Inter-Annotator Agreement

**Symptoms:**
Annotators frequently disagree on same tasks.

**Diagnostic Steps:**

1. **Check if instructions are clear:**
   ```sql
   SELECT expert_instruction FROM project WHERE id = 123;
   ```
   - Vague guidelines → inconsistent annotations

2. **Find most disagreed-upon tasks:**
   ```sql
   SELECT
       t.id,
       t.data,
       COUNT(DISTINCT a.result) as unique_results
   FROM task t
   JOIN task_completion a ON a.task_id = t.id
   WHERE t.project_id = 123
   GROUP BY t.id
   HAVING COUNT(DISTINCT a.result) > 1
   ORDER BY unique_results DESC;
   ```

3. **Check label distribution:**
   ```sql
   SELECT
       result::jsonb->0->'value'->'choices'->>0 as label,
       COUNT(*) as count
   FROM task_completion
   WHERE project_id = 123
   GROUP BY label;
   ```
   - Skewed distribution might indicate confusion

**Common Fixes:**
- Add examples to expert_instruction
- Create golden reference tasks for calibration
- Increase review_percentage temporarily
- Hold team calibration meeting
- Simplify label_config (fewer choices)

---

### Issue 5: Webhook Not Firing

**Symptoms:**
External system not receiving webhook payloads.

**Diagnostic Steps:**

1. **Check webhook is active:**
   ```sql
   SELECT * FROM webhook WHERE id = 789;
   -- is_active must be TRUE
   ```

2. **Check webhook actions configured:**
   ```sql
   SELECT * FROM webhook_action WHERE webhook_id = 789;
   -- Must have matching action (e.g., ANNOTATION_CREATED)
   ```

3. **Check webhook logs (if available):**
   ```sql
   SELECT * FROM webhook_log WHERE webhook_id = 789 ORDER BY created_at DESC LIMIT 10;
   -- Look for HTTP errors (status_code >= 400)
   ```

4. **Test URL manually:**
   ```bash
   curl -X POST https://webhook.url/endpoint \
     -H "Content-Type: application/json" \
     -d '{"test": "payload"}'
   ```

**Common Fixes:**
- Set is_active=TRUE
- Add WebhookAction records
- Fix destination URL (check HTTPS, DNS, firewall)
- Reduce payload size (set send_payload=FALSE)
- Check auth headers (Authorization token expired?)

---

## 🔒 SECURITY NOTES FOR SENSITIVE FIELDS

### Sensitive Fields Requiring Protection

| Field | Table | Risk | Mitigation |
|-------|-------|------|------------|
| **task_data_login** | project | Credentials for external data sources | Encrypt at rest, never log |
| **task_data_password** | project | Paired with task_data_login | Encrypt at rest, mask in UI |
| **token** | project | Invite link security | Rotate on compromise via reset_token() |
| **headers** | webhook | May contain API keys/auth tokens | Encrypt JSONB field, audit access |
| **payload** | project_client_api_configuration | May contain API keys | Sanitize before logging |

### Best Practices

**1. Never Log Sensitive Fields:**
```python
# ❌ WRONG
logger.info(f"Project config: {project.__dict__}")  # Logs task_data_password!

# ✅ RIGHT
safe_dict = {k: v for k, v in project.__dict__.items()
             if k not in ['task_data_login', 'task_data_password', 'token']}
logger.info(f"Project config: {safe_dict}")
```

**2. Mask in API Responses:**
```python
# In serializer
class ProjectSerializer(serializers.ModelSerializer):
    task_data_password = serializers.SerializerMethodField()

    def get_task_data_password(self, obj):
        if obj.task_data_password:
            return "***REDACTED***"
        return None
```

**3. Rotate Tokens Regularly:**
```python
# After security incident
project.reset_token()  # Generates new random hash
project.save()
```

**4. Validate Webhook URLs:**
```python
# Prevent SSRF attacks
def validate_webhook_url(url):
    parsed = urlparse(url)

    # Block internal IPs
    if parsed.hostname in ['localhost', '127.0.0.1', '0.0.0.0']:
        raise ValidationError("Cannot webhook to localhost")

    # Block private IP ranges
    import ipaddress
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        if ip.is_private:
            raise ValidationError("Cannot webhook to private IP")
    except ValueError:
        pass  # Hostname, not IP

    return url
```

**5. Sanitize expert_instruction (XSS Prevention):**
```python
import bleach

ALLOWED_TAGS = ['h1', 'h2', 'h3', 'p', 'ul', 'ol', 'li', 'b', 'i', 'strong', 'em', 'a', 'img']
ALLOWED_ATTRS = {'a': ['href'], 'img': ['src', 'alt']}

project.expert_instruction = bleach.clean(
    user_input,
    tags=ALLOWED_TAGS,
    attributes=ALLOWED_ATTRS,
    strip=True
)
```

---

This concludes the comprehensive enhancements to the Projects & Configuration documentation. These additions cover all requested high-priority, medium-priority, and low-priority items including:

✅ System Overview
✅ User Personas and Use Cases
✅ Complete Enumeration Business Logic
✅ Real-World Scenario Examples
✅ FAQ with SQL Query Examples
✅ Common Pitfalls & Gotchas
✅ State Transition Diagrams
✅ Relationship Diagrams (ASCII)
✅ Onboarding & Learning Path
✅ Historical Context & Design Decisions
✅ Troubleshooting Guide
✅ Security Notes for Sensitive Fields

The document provides detailed, actionable information with concrete code examples, SQL queries, and business logic explanations sourced directly from the codebase analysis.
