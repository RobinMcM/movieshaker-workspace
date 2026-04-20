# MovieShakerV2 Data Model Architecture

## Overview
MovieShakerV2 uses a FastAPI backend with SQLModel (ORM) and PostgreSQL database. The system manages film production workflows with scripts, scenes, characters, projects, and related metadata.

---

## 1. Storage System Architecture

### 1.1 File Storage Location
Files are stored in one of two ways:

**Local Storage:**
- Root: `STORAGE_ROOT` (default: `./storage`)
- Path structure: `{user_id}/{project_id}/{script_id}/`
- Files: `script.pdf`, `script.json`, moodboard images, character images

**DigitalOcean Spaces (S3-compatible):**
- Configured via environment variables:
  - `DO_SPACES_ENDPOINT` (e.g., `fra1.digitaloceanspaces.com`)
  - `DO_SPACES_REGION` (e.g., `fra1`)
  - `DO_SPACES_BUCKET` (bucket name)
  - `DO_SPACES_ACCESS_KEY_ID`
  - `DO_SPACES_SECRET_ACCESS_KEY`
- Uses boto3 S3 client with same key structure as local storage
- Automatic failover: checks for Spaces config, falls back to local

### 1.2 File Organization
```
STORAGE_ROOT/
├── {user_id}/
│   └── {project_id}/
│       ├── {script_id}/
│       │   ├── script.pdf          # Uploaded screenplay
│       │   ├── script.json         # Parsed screenplay (schema v1.0)
│       │   └── (character images)
│       ├── moodboard/
│       │   └── {image_files}       # Generated/uploaded moodboard images
│       └── objects/
│           └── {image_files}       # Generated/uploaded object images
```

### 1.3 Relative Path Format (Used in Database)
```
{user_id}/{project_id}/{script_id}/script.pdf
{user_id}/{project_id}/{script_id}/{filename}.{ext}
```

### 1.4 Script.json Schema
```json
{
  "schema_version": "1.0",
  "metadata": {
    "page_count": 120,
    "title": "Script Name",
    "total_eighths": 80
  },
  "elements": [
    {
      "type": "scene_heading",
      "text": "INT. OFFICE - DAY",
      "eighths": 1,
      "page_number": "1",
      "characters": ["JOHN", "MARY"],
      "scene_number": 1
    },
    {
      "type": "action",
      "text": "John sits at his desk."
    },
    {
      "type": "character",
      "text": "JOHN"
    },
    {
      "type": "dialogue",
      "text": "This is important.",
      "character": "JOHN"
    }
  ]
}
```

---

## 2. Core Database Schema

### 2.1 Project Hierarchy
```
Project (root container)
├── ProjectMember (many)
├── Script (many)
│   ├── Scene (many)
│   │   ├── SceneCharacter (many) [linking to Character]
│   │   ├── TramLine (many)
│   │   │   ├── MoodBoardComposition (many)
│   │   │   ├── MoodBoardImageHistory (many)
│   │   │   ├── MoodBoardVideoHistory (many)
│   │   │   └── MoodBoardCompiledVideo (many)
│   │   └── SceneCost (many)
│   └── Character (many)
├── Budget (one)
│   └── BudgetLineItem (many)
└── SceneCostConfig (one)
```

### 2.2 Project Model
```python
class Project(SQLModel, table=True):
    id: UUID                         # Primary key
    name: str                        # Project name
    description: Optional[str]       # Project description
    status: str                      # "planning" | "pre-production" | "production" | "post-production"
    start_date: Optional[datetime]   # Project start date
    end_date: Optional[datetime]     # Project end date
    director: Optional[str]          # Director name
    film_type: Optional[str]         # "feature" | "short" | "commercial" | "music_video"
    series: Optional[str]            # Series name (if applicable)
    episode: Optional[str]           # Episode number (if applicable)
    aspect_ratio: str                # Default: "16:9"
    creation_method: str             # "standard" | "film_in_a_box"
    created_at: datetime             # Auto-generated
    owner_id: str                    # SuperTokens User ID (creator)
```

### 2.3 Script Model
```python
class Script(SQLModel, table=True):
    id: UUID                         # Primary key
    project_id: UUID                 # FK → Project.id
    user_id: str                     # SuperTokens User ID (uploader)
    name: str                        # Script name/title
    description: Optional[str]       # Optional description
    series: Optional[str]            # Series name
    episode: Optional[str]           # Episode number
    file_path: str                   # Relative path: {user_id}/{project_id}/{script_id}/script.pdf
    is_current: bool                 # Default: False (marks active script in project)
    is_locked: bool                  # Default: False (prevents re-parsing when locked)
    page_count: Optional[int]        # Total pages in PDF
    uploaded_at: datetime            # Auto-generated timestamp
```

### 2.4 Scene Model
```python
class Scene(SQLModel, table=True):
    __tablename__ = "scenes"
    
    id: UUID                         # Primary key
    script_id: UUID                  # FK → Script.id
    user_id: str                     # SuperTokens User ID
    heading: str                     # Scene heading (e.g., "INT. OFFICE - DAY")
    page_number: str                 # Page(s) the scene appears on
    length_in_eighths: Optional[int] # Duration in eighths of a page
    scene_number: Optional[int]      # Sequential scene number
    
    # Scheduling fields (populated during shoot day planning)
    shooting_day: Optional[str]      # Day assigned for shooting
    time_of_day_id: Optional[str]    # "morning" | "afternoon" | "night"
    continuity_day: Optional[int]    # Continuity day counter
    scene_location: Optional[str]    # Primary location
    scene_details: Optional[str]     # Extended description/notes
    location_details: Optional[str]  # Detailed location info
    
    # Cost modifiers (for budget calculations)
    location_type: Optional[str]     # "studio" | "local_interior" | "urban_exterior" | "remote"
    is_night_shoot: Optional[bool]   # True if night shoot affects costs
    has_stunts: Optional[bool]       # True if scene contains stunts
    has_vfx: Optional[bool]          # True if scene has VFX requirements
    extras_count: Optional[int]      # Number of extras needed
    creative_impact: Optional[int]   # 1-5 scale for creative importance
```

### 2.5 Character Model
```python
class Character(SQLModel, table=True):
    __tablename__ = "characters"
    
    id: UUID                         # Primary key
    script_id: UUID                  # FK → Script.id
    user_id: str                     # SuperTokens User ID
    name: str                        # Character/object/scene name
    cast_tier: Optional[str]         # "lead" | "supporting" | "day_player"
    
    # Type determines what this record represents
    type: str                        # "character" (default) | "object" | "scene"
    
    casting_notes: Optional[str]     # Notes for casting director
    character_image_url: Optional[str] # URL to character image (local or Spaces)
    hide_from_view: bool             # False by default; hides from actor role pages if True
    aspect_ratio: Optional[str]      # Image aspect ratio (e.g., "1:1", "16:9")
    series_group: Optional[str]      # Grouping for series management
```

### 2.6 SceneCharacter Linking Table
```python
class SceneCharacter(SQLModel, table=True):
    __tablename__ = "scene_characters"
    
    id: UUID                         # Primary key
    scene_id: UUID                   # FK → Scene.id
    character_id: UUID               # FK → Character.id
    user_id: str                     # SuperTokens User ID
    status: Optional[str]            # Custom status for character in scene
    notes: Optional[str]             # Character-specific notes for this scene
```

### 2.7 ProjectMember Model (Access Control)
```python
class ProjectMember(SQLModel, table=True):
    project_id: UUID                 # FK → Project.id (composite key)
    user_id: str                     # SuperTokens User ID (composite key)
    role: str                        # "owner" | "editor" | "viewer"
    joined_at: datetime              # Timestamp when added to project
```

---

## 3. Data Relationships

### 3.1 Relationship Diagram
```
UserProfile (one per user)
    ↓
Project (multiple per user as owner)
    ├── ProjectMember (defines access control)
    ├── Script (multiple per project)
    │   ├── Scene (multiple per script)
    │   │   ├── SceneCharacter → Character (many-to-many)
    │   │   ├── TramLine (shot coverage)
    │   │   └── SceneCost (budget allocation)
    │   ├── Character (multiple per script)
    │   └── Storage: script.pdf, script.json, images
    ├── Budget (one per project)
    │   └── BudgetLineItem (multiple)
    └── SceneCostConfig (one per project)
```

### 3.2 Access Control Pattern
1. User identified by SuperTokens `user_id`
2. ProjectMember table defines who can access each project
3. All queries verify user is in ProjectMember with appropriate role
4. Roles: "owner" (full access) | "editor" (can modify) | "viewer" (read-only)

### 3.3 Script → Scenes/Characters Flow
```
1. User uploads script.pdf
   ↓
2. POST /scripts/{id}/parse
   ├── Parse PDF → extract scene headings, characters, dialogue
   ├── Clear existing scenes/characters for this script
   ├── Create Scene records (heading, page, length)
   ├── Create Character records (unique character names)
   ├── Create SceneCharacter links (which characters appear in which scenes)
   └── Save script.json (parsed representation)
   ↓
3. Database now contains structured screenplay data
   └── Available for budgeting, scheduling, visualization
```

---

## 4. Complete API Endpoints

### 4.1 Script Management Endpoints

#### GET /scripts/{script_id}
- **Purpose:** Retrieve single script metadata
- **Auth:** Requires session + project membership
- **Response:**
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "project_id": "uuid",
      "name": "My Script",
      "file_url": "user_id/project_id/script_id/script.pdf",
      "uploaded_at": "2026-04-19T...",
      "is_current": true,
      "is_locked": false,
      "page_count": 120,
      "series": null,
      "episode": null,
      "description": null
    }
  }
  ```

#### GET /projects/{project_id}/scripts
- **Purpose:** List all scripts in a project
- **Auth:** Requires session + project membership
- **Caching:** Valkey cache key: `scripts:list:{project_id}` (TTL: 3600s)
- **Response:**
  ```json
  {
    "scripts": [
      { "id": "...", "name": "...", ... }
    ]
  }
  ```

#### POST /projects/{project_id}/scripts
- **Purpose:** Upload new script (PDF)
- **Auth:** Requires session + project membership
- **Method:** Multipart form data
- **Fields:**
  - `file` (required): PDF file (max 50MB)
  - `name` (required): Script name
  - `description` (optional): Description
  - `series` (optional): Series name
  - `episode` (optional): Episode number
- **Behavior:**
  - Saves PDF to storage at `{user_id}/{project_id}/{script_id}/script.pdf`
  - If first script in project, sets `is_current=true`
  - Creates Script record in DB
- **Response:** ScriptResponse with new script details

#### GET /scripts/{script_id}/file
- **Purpose:** Download script PDF or JSON
- **Query Params:**
  - `variant` (optional): "json" to get parsed script.json, omit for PDF
- **Behavior:**
  - For `variant=json`: Returns script.json (error 404 if not parsed yet)
  - Default: Returns script.pdf
  - Works with both local storage and Spaces
- **Response:** File content with appropriate media type

#### POST /scripts/{script_id}/parse
- **Purpose:** Parse PDF/JSON into structured screenplay
- **Auth:** Requires session + project membership
- **Validation:** Script must not be locked (`is_locked=false`)
- **Process:**
  1. Reads script.pdf (or existing script.json)
  2. Extracts scene headings, character names, action, dialogue
  3. Deletes existing Scene/Character records (re-parse)
  4. Creates new Scene records with heading, page, length
  5. Creates new Character records (unique names)
  6. Creates SceneCharacter links
  7. Saves script.json to storage
  8. Updates page_count on Script record
- **Response:**
  ```json
  {
    "success": true,
    "data": {
      "scenes": 45,
      "characters": 12,
      "total_eighths": 75,
      "page_count": 120,
      "schema_version": "1.0"
    }
  }
  ```

#### GET /scripts/{script_id}/scenes
- **Purpose:** List all scenes in script
- **Response:**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "heading": "INT. OFFICE - DAY",
        "page_number": "1",
        "length_in_eighths": 1,
        "scene_number": 1,
        "shooting_day": null,
        "time_of_day_id": null,
        "continuity_day": null,
        "scene_location": null,
        "scene_details": null,
        "location_details": null
      }
    ]
  }
  ```

#### GET /scripts/{script_id}/characters
- **Purpose:** List all characters in script
- **Response:**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "JOHN",
        "script_id": "uuid",
        "type": "character",
        "casting_notes": null,
        "character_image_url": null,
        "hide_from_view": false,
        "aspect_ratio": null,
        "series_group": null
      }
    ]
  }
  ```

#### GET /scripts/{script_id}/stats
- **Purpose:** Get quick stats (scene/character counts)
- **Caching:** Valkey cache key: `scripts:stats:{script_id}` (TTL: 3600s)
- **Response:**
  ```json
  {
    "stats": {
      "scenes": 45,
      "characters": 12,
      "total_eighths": 75
    }
  }
  ```

#### POST /scripts/{script_id}/set-current
- **Purpose:** Mark script as current (active) in project
- **Behavior:** Sets `is_current=true`, all others in project to `false`
- **Response:**
  ```json
  {
    "success": true,
    "message": "Script set as current"
  }
  ```

#### POST /scripts/{script_id}/set-lock
- **Purpose:** Lock/unlock script (prevents re-parsing when locked)
- **Body:**
  ```json
  {
    "is_locked": true
  }
  ```
- **Response:** Updated ScriptResponse

#### PUT /scripts/{script_id}/json
- **Purpose:** Update script.json (full replace)
- **Body:**
  ```json
  {
    "elements": [
      { "type": "scene_heading", "text": "INT. OFFICE - DAY" },
      { "type": "action", "text": "John sits." }
    ],
    "metadata": {
      "page_count": 120
    }
  }
  ```
- **Behavior:** Replaces script.json in storage, does not re-populate DB

#### POST /scripts/{script_id}/json/elements
- **Purpose:** Append elements to script.json (for during-production updates)
- **Body:**
  ```json
  {
    "elements": [
      { "type": "action", "text": "New scene during production" }
    ]
  }
  ```
- **Behavior:** Appends to existing script.json

#### DELETE /scripts/{script_id}
- **Purpose:** Delete script and all associated data
- **Cascade:** Deletes PDF, JSON, Scene, Character, SceneCharacter records
- **Response:**
  ```json
  {
    "success": true
  }
  ```

### 4.2 Scene Management Endpoints

#### PUT /scenes/{scene_id}
- **Purpose:** Update single scene
- **Body:** Any of:
  ```json
  {
    "shooting_day": "Monday",
    "time_of_day_id": "morning",
    "continuity_day": 1,
    "scene_location": "Studio A",
    "scene_details": "Indoor scene with extras",
    "location_details": "Studio A, Stage 3"
  }
  ```
- **Response:** Updated SceneResponse

#### PUT /scenes/bulk-update
- **Purpose:** Update multiple scenes at once
- **Body:**
  ```json
  {
    "scene_ids": ["uuid1", "uuid2", "uuid3"],
    "updates": {
      "shooting_day": "Monday",
      "scene_location": "Downtown LA"
    }
  }
  ```
- **Allowed fields:** `shooting_day`, `time_of_day_id`, `continuity_day`, `scene_location`, `scene_details`, `location_details`
- **Response:**
  ```json
  {
    "success": true,
    "message": "Scenes updated"
  }
  ```

### 4.3 Character Management Endpoints

#### GET /scripts/{script_id}/characters
- **Purpose:** List characters (see endpoint details in Scripts section)

#### POST /scripts/{script_id}/characters
- **Purpose:** Create new character manually
- **Body:**
  ```json
  {
    "name": "JOHN",
    "type": "character",
    "casting_notes": "Lead role",
    "aspect_ratio": "16:9",
    "series_group": null
  }
  ```
- **Response:** CharacterResponse

#### PUT /characters/{character_id}
- **Purpose:** Update character details
- **Body:** Any of:
  ```json
  {
    "casting_notes": "Updated notes",
    "aspect_ratio": "1:1",
    "hide_from_view": false,
    "character_image_url": "https://example.com/image.jpg"
  }
  ```
- **Response:** Updated character data

#### DELETE /characters/{character_id}
- **Purpose:** Delete character
- **Response:**
  ```json
  {
    "success": true,
    "message": "Deleted"
  }
  ```

#### POST /characters/{character_id}/generate-image
- **Purpose:** Generate character image via AI (FAL models through gateway)
- **Auth:** Requires credit balance
- **Body:**
  ```json
  {
    "prompt": "Professional headshot, serious expression",
    "aspect_ratio": "1:1",
    "model": "model-id",
    "dry_run": false
  }
  ```
- **Response:** Updated character with new `character_image_url`

### 4.4 Scene-Character Linking Endpoints

#### GET /scripts/{script_id}/scene-characters
- **Purpose:** List all scene-character links in script
- **Response:**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "scene_id": "uuid",
        "character_id": "uuid",
        "status": null,
        "notes": null
      }
    ]
  }
  ```

#### PUT /scripts/scene-characters/{scene_character_id}
- **Purpose:** Update scene-character link
- **Body:**
  ```json
  {
    "status": "confirmed",
    "notes": "Character wears costume in this scene"
  }
  ```
- **Response:** Updated SceneCharacterResponse

### 4.5 Public Actor Role Endpoints

#### GET /public/actor-role/{project_id}/{script_id}/{character_id}
- **Purpose:** Public-facing endpoint for actors to view their role
- **Auth:** No authentication required
- **Access Control:** Only visible if character has `hide_from_view=false`
- **Response:**
  ```json
  {
    "character": {
      "name": "JOHN",
      "character_image_url": "..."
    },
    "project": "My Film",
    "script": "Script Title",
    "scenes": [
      {
        "id": "uuid",
        "scene_number": 1,
        "heading": "INT. OFFICE - DAY",
        "description": "Scene details...",
        "page_number": "1"
      }
    ],
    "script_elements": [
      {
        "type": "character",
        "text": "JOHN",
        "character": null
      },
      {
        "type": "dialogue",
        "text": "This is important.",
        "character": "JOHN"
      }
    ]
  }
  ```

### 4.6 Storage Endpoints

#### GET /api/storage/{path}
- **Purpose:** Serve stored images (moodboard, character images)
- **Allowed Paths:**
  - Legacy: `moodboard/{user_id}/...` or `objects/{user_id}/...`
  - Project-scoped: `{user_id}/{project_id}/.../moodboard/...` or `.../objects/...`
- **Auth:** Requires session; verifies user_id in path
- **Response:** Image file with appropriate Content-Type

---

## 5. Database Migrations

### 5.1 Auto-migrations in db.py

The system includes idempotent migrations:

```python
_migrate_user_profile_roles()           # Add role, producer_tier, blocked
_migrate_user_profile_drop_admin()      # Remove legacy admin column
_migrate_user_profile_email_verified()  # Add email_verified_at
_migrate_project_table()                # Add status, dates, metadata
_migrate_script_table()                 # Add is_locked, page_count
_migrate_scene_cost_modifiers()         # Add location_type, is_night_shoot, etc.
```

---

## 6. Caching Strategy

### 6.1 Cache Keys (Valkey)

```python
scripts:list:{project_id}      # TTL: 3600s
scripts:stats:{script_id}      # TTL: 3600s
projects:list:{user_id}        # TTL: 3600s
```

### 6.2 Cache Invalidation
- Automatically cleared when data changes
- E.g., `POST /scripts` clears `scripts:list:{project_id}`

---

## 7. Data Flow Examples

### 7.1 Script Upload and Parse Flow
```
1. User uploads script.pdf via POST /projects/{project_id}/scripts
   ├── Save PDF to storage: {user_id}/{project_id}/{script_id}/script.pdf
   ├── Create Script record (is_current=true if first)
   └── Return ScriptResponse

2. User calls POST /scripts/{script_id}/parse
   ├── Read script.pdf from storage
   ├── Extract scenes, characters, dialogue
   ├── Delete existing Scene/Character records
   ├── Create new records from parsed data
   ├── Save script.json to storage
   └── Return parse stats (scene count, character count, etc.)

3. User accesses script via GET /scripts/{script_id}/scenes
   ├── Query Scene records for this script
   └── Return scene list

4. User accesses characters via GET /scripts/{script_id}/characters
   ├── Query Character records for this script
   └── Return character list
```

### 7.2 Scene Scheduling Flow
```
1. User calls PUT /scenes/{scene_id} with scheduling info
   ├── Update: shooting_day, time_of_day_id, continuity_day
   └── Persist to DB

2. Or bulk update via PUT /scenes/bulk-update
   ├── Update multiple scenes with same values
   └── Useful for assigning location to multiple scenes

3. User calls GET /scripts/{script_id}/scenes
   ├── Returns all scenes with their scheduling info
   └── Frontend visualizes on tram lines or calendar
```

### 7.3 Character Image Generation Flow
```
1. User calls POST /characters/{character_id}/generate-image
   ├── Check user has AI credits
   ├── Resolve model ID from catalog
   ├── Send request to openrouter-gateway → FAL model
   ├── Wait for image URL in response
   ├── Save image to storage
   ├── Update Character.character_image_url
   └── Return updated character

2. Image stored at: {user_id}/{project_id}/.../character_images/{character_id}.{ext}
   
3. Later, user retrieves image via GET /api/storage/...
   ├── Verify access (user matches)
   └── Stream image to client
```

---

## 8. Key Design Patterns

### 8.1 Access Control
- **Pattern:** Verify ProjectMember exists before any operation
- **Implementation:** `_ensure_project_member()` helper
- **Fallback:** Script lookup verifies project membership

### 8.2 File Storage Abstraction
- **Pattern:** Supports both local and cloud (Spaces) storage
- **Implementation:** `uses_spaces()` check, identical key format
- **Fallback:** Automatic: tries Spaces if configured, uses local storage otherwise

### 8.3 Idempotent Migrations
- **Pattern:** ALTER TABLE ... ADD COLUMN IF NOT EXISTS
- **Benefit:** Safe to run multiple times, no errors on existing columns

### 8.4 Cache Invalidation
- **Pattern:** Delete cache when data changes
- **Benefit:** Ensures consistency without complex cache TTL logic

### 8.5 Transactional Parsing
- **Pattern:** Delete all related records before re-parsing
- **Benefit:** Clean slate for parse results, prevents duplicates

---

## 9. Summary Table: CRUD Operations

| Entity | Create | Read | Update | Delete |
|--------|--------|------|--------|--------|
| Script | POST /projects/{pid}/scripts | GET /scripts/{id} GET /projects/{pid}/scripts | POST /scripts/{id}/set-current POST /scripts/{id}/set-lock | DELETE /scripts/{id} |
| Scene | AUTO (via parse) | GET /scripts/{sid}/scenes | PUT /scenes/{id} PUT /scenes/bulk-update | AUTO (on re-parse) |
| Character | POST /scripts/{sid}/characters AUTO (via parse) | GET /scripts/{sid}/characters | PUT /characters/{id} | DELETE /characters/{id} |
| SceneCharacter | AUTO (via parse) | GET /scripts/{sid}/scene-characters | PUT /scripts/scene-characters/{id} | AUTO (on re-parse) |
| ScriptJSON | AUTO (via parse) | GET /scripts/{id}/file?variant=json | PUT /scripts/{id}/json POST /scripts/{id}/json/elements | N/A |

---

## 10. Environment Configuration

### Required Environment Variables
```bash
# Database
DATABASE_URL=postgresql://user:password@db:5432/movieshaker

# Storage
STORAGE_ROOT=./storage

# Optional: DigitalOcean Spaces
DO_SPACES_ENDPOINT=fra1.digitaloceanspaces.com
DO_SPACES_REGION=fra1
DO_SPACES_BUCKET=bucket-name
DO_SPACES_ACCESS_KEY_ID=key
DO_SPACES_SECRET_ACCESS_KEY=secret

# Gateway (AI models)
GATEWAY_BASE_URL=http://openrouter-gateway:5000
GATEWAY_INTERNAL_API_KEY=key

# Auth
SUPERTOKENS_CONNECTION_URI=http://supertokens:3567

# Server
API_BASE_URL=http://localhost:8000
WEBSITE_DOMAIN=http://localhost:3000
```

---

## 11. Important Constraints & Limitations

1. **Script Locking:** Once locked (`is_locked=true`), a script cannot be re-parsed
2. **Parse Destructive:** Re-parsing deletes all existing Scene/Character records
3. **Spaces Key Format:** Must match local storage format exactly
4. **File Size Limit:** 50MB per script file
5. **PDF/JSON Support:** Parse endpoint supports both PDF and JSON formats
6. **User Isolation:** All queries filtered by `user_id` for data privacy

---

## 12. Related Systems

### External Dependencies
- **SuperTokens:** Authentication and session management
- **openrouter-gateway:** AI image generation (FAL models)
- **PostgreSQL:** Primary data store
- **Valkey:** Distributed cache
- **DigitalOcean Spaces (optional):** File storage alternative

### Related Features
- **Tram Lines:** Shot coverage markers per scene
- **Budget:** Project-level budgeting with line items
- **Scene Costs:** Per-scene cost calculations
- **Moodboard:** Visual development with generated/uploaded images

