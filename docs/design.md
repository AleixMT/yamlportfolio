# Design Document — yamlresume feature extensions

This document describes the architecture for three planned features on top of
the upstream `yamlresume` codebase. Read it alongside `CLAUDE.md` when
implementing any of these features.

---

## 1. Multilingual Content

### Problem

The current `locale.language` setting translates **output labels** (section
headings, degree names, fluency levels) into one of 11 supported locales.
Resume **content** (summaries, descriptions, names) is always a single,
monolingual string — users who want to distribute their CV in multiple languages
must maintain separate YAML files.

### Goal

Allow any text field in the resume content to hold translations for multiple
locales inside a single YAML file. Building the resume with `locale.language:
es` should produce a fully Spanish document without requiring a separate file.

### New type: `MultilingualString`

```typescript
// packages/core/src/models/types/options.ts (or a new primitives file)
type MultilingualString = string | Partial<Record<LocaleLanguage, string>>
```

A plain `string` value is treated as locale-agnostic (displayed as-is in all
locales). An object value maps locale codes to per-locale text.

**Example YAML**:
```yaml
content:
  basics:
    name: John Doe
    summary:
      en: Software engineer with 10 years of experience.
      es: Ingeniero de software con 10 años de experiencia.
      fr: Ingénieur logiciel avec 10 ans d'expérience.
  work:
    - name: Acme Corp
      position: Senior Engineer
      startDate: "2020-01"
      summary:
        en: Led the backend team.
        es: Lideré el equipo de backend.
```

### Affected fields

Every user-authored text field becomes `MultilingualString`. Required and enum
fields (`degree`, `level`, `fluency`, `network`, dates, URLs) remain `string`
because they are already translated via the options translation system.

Fields upgraded to `MultilingualString`:

| Section | Fields |
|---|---|
| `basics` | `headline`, `summary` |
| `work` | `position`, `summary` |
| `education` | `area`, `summary` |
| `volunteer` | `position`, `summary` |
| `awards` | `title`, `summary` |
| `certificates` | `name` |
| `publications` | `name`, `summary` |
| `skills` | `name` |
| `interests` | `name` |
| `references` | `summary` |
| `projects` | `name`, `description`, `summary` |
| `location` | `address`, `region` |

`name` fields for organizations/institutions/companies stay as `string` (proper
nouns are not translated).

### Zod schema changes

Add a shared primitive to
`packages/core/src/schema/primitives.ts`:

```typescript
export const MultilingualStringSchema = z.union([
  z.string(),
  z.record(LocaleLanguageSchema, z.string()),
])
```

Update per-section schemas to use `MultilingualStringSchema` instead of
`z.string()` for the fields listed above.

### Preprocess step: `resolveMultilingualStrings`

Add to `packages/core/src/preprocess/transform.ts`:

```typescript
function resolveMultilingualString(
  value: MultilingualString | undefined,
  language: LocaleLanguage
): string | undefined {
  if (value == null || typeof value === 'string') return value
  return value[language] ?? value['en'] ?? Object.values(value)[0]
}

export function resolveMultilingualStrings(resume: Resume): Resume {
  // Walk all content fields and replace MultilingualString objects
  // with the resolved plain string for resume.locale.language.
  // Returns a structurally identical resume with only plain strings.
}
```

This step runs **first** in the preprocessing pipeline so all subsequent
transforms receive plain strings.

### Backward compatibility

A YAML file with only plain strings continues to work unchanged. The Zod schema
accepts both forms.

---

## 2. Projects with IDs

### Problem

The existing `projects` section and the `work`, `awards`, `volunteer`, and
`certificates` sections are independent. There is no way to express "this job
involved these projects" or "this award was tied to this project" at the data
level.

### Goal

Give each project entry a stable `id`. Other section items (`work`, `awards`,
`volunteer`, `certificates`) can list project IDs to declare that those projects
belong to that entry. Renderers can then render the referenced projects inline
under the parent entry.

### Type changes (`packages/core/src/models/types/content.ts`)

```typescript
type ProjectItem = {
  name: string
  startDate: string
  summary: string

  id?: string           // NEW — unique identifier, kebab-case recommended
  description?: string
  endDate?: string
  keywords?: Keywords
  url?: string

  computed?: {
    dateRange: string
    keywords: string
    startDate: string
    endDate: string
    summary: string
  }
}

type WorkItem = {
  // existing fields …
  projects?: string[]   // NEW — list of ProjectItem.id values
  computed?: {
    // existing computed fields …
    projects?: ProjectItem[]  // NEW — resolved project items
  }
}

type AwardItem = {
  // existing fields …
  projects?: string[]   // NEW
  computed?: {
    // existing computed fields …
    projects?: ProjectItem[]  // NEW
  }
}

type VolunteerItem = {
  // existing fields …
  projects?: string[]   // NEW
  computed?: {
    // existing computed fields …
    projects?: ProjectItem[]  // NEW
  }
}

type CertificateItem = {
  // existing fields …
  projects?: string[]   // NEW
  computed?: {
    // existing computed fields …
    projects?: ProjectItem[]  // NEW
  }
}
```

### Zod schema changes

```typescript
// packages/core/src/schema/content/projects.ts
export const ProjectIdSchema = z.string().regex(/^[a-z0-9-]+$/).optional()

export const ProjectItemSchema = z.object({
  id: ProjectIdSchema,
  name: ...,
  // rest unchanged
})
```

Add `projects: z.array(z.string()).nullish()` to all four consumer schemas:

```typescript
// packages/core/src/schema/content/work.ts
export const WorkItemSchema = z.object({ /* … */ projects: z.array(z.string()).nullish() })

// packages/core/src/schema/content/awards.ts
export const AwardItemSchema = z.object({ /* … */ projects: z.array(z.string()).nullish() })

// packages/core/src/schema/content/volunteer.ts
export const VolunteerItemSchema = z.object({ /* … */ projects: z.array(z.string()).nullish() })

// packages/core/src/schema/content/certificates.ts
export const CertificateItemSchema = z.object({ /* … */ projects: z.array(z.string()).nullish() })
```

Validation should warn (not error) if a referenced project ID does not exist in
`content.projects`.

### Preprocess step: `resolveProjectRefs`

```typescript
export function resolveProjectRefs(resume: Resume): Resume {
  const projectMap = new Map(
    (resume.content.projects ?? [])
      .filter(p => p.id)
      .map(p => [p.id!, p])
  )

  const resolveProjects = (ids: string[] | undefined) =>
    ids?.map(id => projectMap.get(id)).filter(Boolean) ?? []

  return {
    ...resume,
    content: {
      ...resume.content,
      work: resume.content.work?.map(item => ({
        ...item,
        computed: { ...item.computed, projects: resolveProjects(item.projects) },
      })),
      awards: resume.content.awards?.map(item => ({
        ...item,
        computed: { ...item.computed, projects: resolveProjects(item.projects) },
      })),
      volunteer: resume.content.volunteer?.map(item => ({
        ...item,
        computed: { ...item.computed, projects: resolveProjects(item.projects) },
      })),
      certificates: resume.content.certificates?.map(item => ({
        ...item,
        computed: { ...item.computed, projects: resolveProjects(item.projects) },
      })),
    },
  }
}
```

### Rendering

Renderers check `computed.projects` on each work/awards/volunteer/certificates
item. If present, they render a sub-list of project names (and optionally
summaries) below the main entry description.

The standalone `projects` section continues to render all projects as before.

### ID uniqueness constraint

`resolveProjectRefs` (or the Zod schema's `superRefine`) should log a warning
if:
- Two `ProjectItem` entries share the same `id`
- A referenced ID is not found in `content.projects`

---

## 3. Skill Entities with IDs

### Problem

The existing `skills` section is a flat list of skill entries (name + level +
keywords). There is no way to tag an experience entry with the specific skills
it involved. Users who want to show "which technologies I used where" must
duplicate the information in free-text `keywords` or `summary` fields.

### Goal

Give each skill entry a stable `id`. Any project section item
(`projects`, `work`, `awards`, `volunteer`, `certificates`) can then declare
which skill IDs it involved. Renderers render the resolved skill names (and
optionally levels) alongside each entry.

### Type changes (`packages/core/src/models/types/content.ts`)

```typescript
type SkillItem = {
  level: Level
  name: string

  id?: string         // NEW — unique identifier, kebab-case recommended
  keywords?: Keywords

  computed?: {
    level: string
    keywords: string
  }
}

// Skills can be referenced from any experience-bearing section item.
// The computed.skills field holds the resolved SkillItem objects for renderers.

type ProjectItem = {
  // existing fields …
  skills?: string[]          // NEW — list of SkillItem.id values
  computed?: {
    // existing computed fields …
    skills?: SkillItem[]     // NEW — resolved skill items
  }
}

type WorkItem = {
  // existing fields …
  skills?: string[]          // NEW
  computed?: {
    // existing computed fields …
    skills?: SkillItem[]     // NEW
  }
}

type AwardItem = {
  // existing fields …
  skills?: string[]          // NEW
  computed?: {
    // existing computed fields …
    skills?: SkillItem[]     // NEW
  }
}

type VolunteerItem = {
  // existing fields …
  skills?: string[]          // NEW
  computed?: {
    // existing computed fields …
    skills?: SkillItem[]     // NEW
  }
}

type CertificateItem = {
  // existing fields …
  skills?: string[]          // NEW
  computed?: {
    // existing computed fields …
    skills?: SkillItem[]     // NEW
  }
}
```

The same `skills?: string[]` pattern is applied to all five section types
(`projects`, `work`, `awards`, `volunteer`, `certificates`) so that a user can
tag any experience entry with the skills/technologies it involved.

### Zod schema changes

```typescript
// packages/core/src/schema/content/skills.ts
export const SkillIdSchema = z.string().regex(/^[a-z0-9-]+$/).optional()

export const SkillItemSchema = z.object({
  id: SkillIdSchema,
  level: LevelOptionSchema,
  name: SkillNameSchema,
  keywords: nullifySchema(KeywordsSchema),
})
```

Add `skills: z.array(z.string()).nullish()` to all five schemas:

```typescript
// packages/core/src/schema/content/projects.ts
export const ProjectItemSchema = z.object({ /* … */ skills: z.array(z.string()).nullish() })

// packages/core/src/schema/content/work.ts
export const WorkItemSchema = z.object({ /* … */ skills: z.array(z.string()).nullish() })

// packages/core/src/schema/content/awards.ts
export const AwardItemSchema = z.object({ /* … */ skills: z.array(z.string()).nullish() })

// packages/core/src/schema/content/volunteer.ts
export const VolunteerItemSchema = z.object({ /* … */ skills: z.array(z.string()).nullish() })

// packages/core/src/schema/content/certificates.ts
export const CertificateItemSchema = z.object({ /* … */ skills: z.array(z.string()).nullish() })
```

### Preprocess step: `resolveSkillRefs`

The step maps skill IDs to resolved `SkillItem` objects and injects them into
`computed.skills` on every experience-bearing section item.

```typescript
export function resolveSkillRefs(resume: Resume): Resume {
  const skillMap = new Map(
    (resume.content.skills ?? [])
      .filter(s => s.id)
      .map(s => [s.id!, s])
  )

  const resolveSkills = (ids: string[] | undefined): SkillItem[] =>
    ids?.map(id => skillMap.get(id)).filter(Boolean) ?? []

  return {
    ...resume,
    content: {
      ...resume.content,
      projects: resume.content.projects?.map(item => ({
        ...item,
        computed: { ...item.computed, skills: resolveSkills(item.skills) },
      })),
      work: resume.content.work?.map(item => ({
        ...item,
        computed: { ...item.computed, skills: resolveSkills(item.skills) },
      })),
      awards: resume.content.awards?.map(item => ({
        ...item,
        computed: { ...item.computed, skills: resolveSkills(item.skills) },
      })),
      volunteer: resume.content.volunteer?.map(item => ({
        ...item,
        computed: { ...item.computed, skills: resolveSkills(item.skills) },
      })),
      certificates: resume.content.certificates?.map(item => ({
        ...item,
        computed: { ...item.computed, skills: resolveSkills(item.skills) },
      })),
    },
  }
}
```

### Rendering

When rendering any of these five section items, if `computed.skills` is
non-empty, append a "Technologies:" or "Skills:" line listing the resolved
skill names. The label is translated via the existing `getTemplateTranslations`
mechanism (add a new `skills` term entry).

### Full example YAML

Both `projects` and `skills` cross-references can appear on `work`, `awards`,
`volunteer`, and `certificates` entries, as well as on standalone `projects`.

```yaml
locale:
  language: en

content:
  basics:
    name: Alex Developer

  skills:
    - id: typescript
      name: TypeScript
      level: Expert
    - id: react
      name: React
      level: Advanced
    - id: postgres
      name: PostgreSQL
      level: Intermediate
    - id: leadership
      name: Team Leadership
      level: Advanced

  projects:
    - id: portfolio-site
      name: Personal Portfolio
      startDate: "2024-01"
      summary: Built a personal website to showcase work.
      skills:
        - typescript
        - react
    - id: billing-service
      name: Billing Microservice
      startDate: "2023-06"
      endDate: "2024-01"
      summary: Designed and built a billing service from scratch.
      skills:
        - typescript
        - postgres

  work:
    - name: Acme Corp
      position: Senior Engineer
      startDate: "2022-01"
      summary: Full-stack development across multiple products.
      projects:
        - billing-service
      skills:
        - typescript
        - postgres
        - leadership

  awards:
    - title: Open Source Contributor of the Year
      awarder: Linux Foundation
      date: "2023-11"
      summary: Recognised for sustained contributions to the ecosystem.
      projects:
        - portfolio-site
      skills:
        - react
        - typescript

  volunteer:
    - organization: Open Source Initiative
      position: Contributor
      startDate: "2021-03"
      summary: Maintained a React component library used by 200+ projects.
      projects:
        - portfolio-site
      skills:
        - react
        - typescript

  certificates:
    - name: AWS Solutions Architect
      issuer: Amazon Web Services
      date: "2023-06"
      projects:
        - billing-service
      skills:
        - postgres
```

---

## 4. Universal IDs and Cross-References

### Problem

The existing `projects` and `skills` features (§2 and §3) introduced IDs only
for those two section types, with fixed-direction references (e.g. `work` lists
project IDs). This is too narrow: users may want to express that a skill is
related to an award, or that a project is connected to a publication or a
volunteer experience. There is also no way to link any two arbitrary resume
entries.

### Goal

- Give **every experience-bearing section item** a stable, optional `id` field:
  `work`, `volunteer`, `projects`, `references`, `publications`, `certificates`,
  `awards`, `education`, and `skills`.
- Add a `relatedTo?: string[]` field to **`ProjectItem`** and **`SkillItem`**
  that can reference the `id` of **any entity** across the entire resume.
- A preprocess step resolves the string IDs to their typed entities and stores
  them in `computed.relatedTo`.

### New type: `RelatedEntity`

```typescript
// packages/core/src/models/types/content.ts
type RelatedEntity =
  | { section: 'awards';        item: AwardItem }
  | { section: 'certificates';  item: CertificateItem }
  | { section: 'education';     item: EducationItem }
  | { section: 'projects';      item: ProjectItem }
  | { section: 'publications';  item: PublicationItem }
  | { section: 'references';    item: ReferenceItem }
  | { section: 'skills';        item: SkillItem }
  | { section: 'volunteer';     item: VolunteerItem }
  | { section: 'work';          item: WorkItem }
```

### Type changes (`packages/core/src/models/types/content.ts`)

Every section item listed gains an optional `id` field:

```typescript
type WorkItem        = { id?: string; /* existing fields */ }
type VolunteerItem   = { id?: string; /* existing fields */ }
type AwardItem       = { id?: string; /* existing fields */ }
type CertificateItem = { id?: string; /* existing fields */ }
type EducationItem   = { id?: string; /* existing fields */ }
type PublicationItem = { id?: string; /* existing fields */ }
type ReferenceItem   = { id?: string; /* existing fields */ }
type SkillItem       = { id?: string; /* existing fields */ }
```

`ProjectItem` gains both `id` and `relatedTo`, and its `computed` block gains
`relatedTo`:

```typescript
type ProjectItem = {
  id?: string
  relatedTo?: string[]    // NEW — list of any entity IDs across the resume
  computed?: {
    // existing computed fields …
    relatedTo?: RelatedEntity[]  // NEW — resolved entities
  }
}
```

`SkillItem` gains the same:

```typescript
type SkillItem = {
  id?: string
  relatedTo?: string[]    // NEW — list of any entity IDs across the resume
  computed?: {
    // existing computed fields …
    relatedTo?: RelatedEntity[]  // NEW — resolved entities
  }
}
```

### Zod schema: `EntityIdSchema`

Add a shared primitive to `packages/core/src/schema/primitives.ts`:

```typescript
export const EntityIdSchema = z
  .string()
  .regex(/^[a-z0-9-]+$/, { message: 'id must be lowercase alphanumeric with hyphens.' })
  .meta({ title: 'ID', description: 'A unique entity identifier (kebab-case).' })
```

Add `id: EntityIdSchema.nullish()` to all nine item schemas and
`relatedTo: z.array(z.string()).nullish()` to `ProjectItemSchema` and
`SkillItemSchema`.

### Preprocess step: `resolveRelatedRefs`

```typescript
export function resolveRelatedRefs(resume: Resume): Resume {
  // Build a global id → { section, item } map from all nine section types.
  const entityMap = new Map<string, RelatedEntity>()
  const sectionEntries: Array<{
    section: RelatedEntity['section']
    items: Array<{ id?: string }>
  }> = [
    { section: 'awards',        items: resume.content.awards        ?? [] },
    { section: 'certificates',  items: resume.content.certificates  ?? [] },
    { section: 'education',     items: resume.content.education      ?? [] },
    { section: 'projects',      items: resume.content.projects       ?? [] },
    { section: 'publications',  items: resume.content.publications   ?? [] },
    { section: 'references',    items: resume.content.references     ?? [] },
    { section: 'skills',        items: resume.content.skills         ?? [] },
    { section: 'volunteer',     items: resume.content.volunteer      ?? [] },
    { section: 'work',          items: resume.content.work           ?? [] },
  ]

  for (const { section, items } of sectionEntries) {
    for (const item of items) {
      if (item.id) {
        entityMap.set(item.id, { section, item } as RelatedEntity)
      }
    }
  }

  const resolveIds = (ids: string[] | undefined): RelatedEntity[] =>
    (ids ?? []).map(id => entityMap.get(id)).filter(Boolean) as RelatedEntity[]

  return {
    ...resume,
    content: {
      ...resume.content,
      projects: resume.content.projects?.map(item => ({
        ...item,
        computed: { ...item.computed, relatedTo: resolveIds(item.relatedTo) },
      })),
      skills: resume.content.skills?.map(item => ({
        ...item,
        computed: { ...item.computed, relatedTo: resolveIds(item.relatedTo) },
      })),
    },
  }
}
```

### ID uniqueness

`resolveRelatedRefs` should emit a console warning (not throw) when:
- Two entities in the same or different sections share the same `id`.
- A `relatedTo` entry references an ID that does not exist in any section.

### Full example YAML

```yaml
locale:
  language: en

content:
  basics:
    name: Alex Developer

  skills:
    - id: typescript
      name: TypeScript
      level: Expert
      relatedTo:
        - billing-service
        - acme-work

    - id: react
      name: React
      level: Advanced
      relatedTo:
        - portfolio-site

  projects:
    - id: portfolio-site
      name: Personal Portfolio
      startDate: "2024-01"
      summary: Built a personal website to showcase work.
      relatedTo:
        - typescript
        - react
        - open-source-award

    - id: billing-service
      name: Billing Microservice
      startDate: "2023-06"
      endDate: "2024-01"
      summary: Designed and built a billing service from scratch.
      relatedTo:
        - typescript
        - acme-work

  work:
    - id: acme-work
      name: Acme Corp
      position: Senior Engineer
      startDate: "2022-01"
      summary: Full-stack development across multiple products.

  awards:
    - id: open-source-award
      title: Open Source Contributor of the Year
      awarder: Linux Foundation
      date: "2023-11"
      summary: Recognised for sustained contributions.

  education:
    - id: cs-degree
      area: Computer Science
      institution: State University
      degree: Bachelor
      startDate: "2016-09"
      endDate: "2020-05"

  volunteer:
    - id: osi-volunteer
      organization: Open Source Initiative
      position: Contributor
      startDate: "2021-03"
      summary: Maintained a React component library.

  certificates:
    - id: aws-cert
      name: AWS Solutions Architect
      issuer: Amazon Web Services
      date: "2023-06"

  publications:
    - id: my-paper
      name: Serverless Billing at Scale
      publisher: ACM
      releaseDate: "2024-03"
      summary: Published research on billing service architecture.

  references:
    - id: alice-ref
      name: Alice Manager
      summary: Excellent engineer with strong leadership skills.
```

---

## Preprocess pipeline order

After all four features are implemented, the transform pipeline runs in this
order:

1. `resolveMultilingualStrings` — replace `MultilingualString` with plain strings
2. `resolveRelatedRefs` — build global entity map; inject `computed.relatedTo`
   on projects and skills
3. (existing transforms) `normalizeResumeContentSections`, `transformDate`, …

Step 2 must run after step 1 so that entity fields are plain strings when
stored in `computed.relatedTo`.

---

## Checklist for implementing each feature

### Multilingual content
- [ ] Add `MultilingualString` type to `models/types/options.ts`
- [ ] Add `multilingualStringSchema` to `schema/primitives.ts`
- [ ] Update affected field types in `models/types/content.ts`
- [ ] Update affected Zod schemas in `schema/content/`
- [ ] Add `resolveMultilingualStrings` to `preprocess/transform.ts`
- [ ] Wire up as first step in the pipeline
- [ ] Update `RESUME_SECTION_ITEMS` defaults to use plain strings (no change needed)
- [ ] Add unit tests for `resolveMultilingualString`
- [ ] Add schema tests for the union type
- [ ] Update `schema.json` via `pnpm core build`

### Projects with IDs
- [ ] Add `id?: string` to `ProjectItem` in `models/types/content.ts`
- [ ] Add `projects?: string[]` + `computed.projects` to `WorkItem`, `AwardItem`,
      `VolunteerItem`, `CertificateItem`
- [ ] Add `ProjectIdSchema` to `schema/content/projects.ts`
- [ ] Add `projects` field to work/awards/volunteer/certificates Zod schemas
- [ ] Add `resolveProjectRefs` to `preprocess/transform.ts`
- [ ] Update LaTeX, HTML, Markdown renderers to render `computed.projects` for
      all four consumer section types
- [ ] Add unit tests for `resolveProjectRefs`
- [ ] Add renderer smoke tests
- [ ] Update `schema.json`

### Skill entities with IDs
- [ ] Add `id?: string` to `SkillItem` in `models/types/content.ts`
- [ ] Add `skills?: string[]` + `computed.skills` to `ProjectItem`, `WorkItem`,
      `AwardItem`, `VolunteerItem`, `CertificateItem`
- [ ] Add `SkillIdSchema` to `schema/content/skills.ts`
- [ ] Add `skills: z.array(z.string()).nullish()` to project/work/awards/volunteer/
      certificates Zod schemas
- [ ] Add `resolveSkillRefs` to `preprocess/transform.ts` (covers all five
      section types)
- [ ] Add `skills` term to `getTemplateTranslations` in `translations/template.ts`
- [ ] Update LaTeX, HTML, Markdown renderers to render `computed.skills` for all
      five section types
- [ ] Add unit tests for `resolveSkillRefs`
- [ ] Add renderer smoke tests
- [ ] Update `schema.json`

### Universal IDs and cross-references
- [ ] Add `RelatedEntity` discriminated union type to `models/types/content.ts`
- [ ] Add `id?: string` to `AwardItem`, `CertificateItem`, `EducationItem`,
      `PublicationItem`, `ProjectItem`, `ReferenceItem`, `SkillItem`,
      `VolunteerItem`, `WorkItem` in `models/types/content.ts`
- [ ] Add `relatedTo?: string[]` + `computed.relatedTo?: RelatedEntity[]` to
      `ProjectItem` and `SkillItem`
- [ ] Add `EntityIdSchema` to `schema/primitives.ts`
- [ ] Add `id: EntityIdSchema.nullish()` to all nine item schemas
- [ ] Add `relatedTo: z.array(z.string()).nullish()` to `ProjectItemSchema` and
      `SkillItemSchema`
- [ ] Add `resolveRelatedRefs` to `preprocess/transform.ts`
- [ ] Wire `resolveRelatedRefs` as step 2 in the pipeline (after
      `resolveMultilingualStrings`)
- [ ] Add unit tests for `resolveRelatedRefs`
- [ ] Add schema tests for `id` and `relatedTo` in each section
- [ ] Update `schema.json` via `pnpm core build`
- [ ] Create `docs/entity_diagram.puml` entity relationship diagram
