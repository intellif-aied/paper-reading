# XSkill Evolution

This context describes how coding-agent work becomes reusable, versioned Skills and how those Skills return to individual or team harnesses.

## Capture

**Ecosystem**:
A supported harness family with known session-source and Skill-installation conventions.
_Avoid_: Agent, provider

**Native Session**:
The harness-owned interaction record before XSkill normalization.
_Avoid_: Trajectory

**Bridge**:
The normalized boundary that turns a Native Session into an XSkill Trajectory while retaining attribution metadata.
_Avoid_: Upload, copy

**Trajectory**:
The durable, canonical record of one agent session used as learning evidence.
_Avoid_: Native Session, Atom

**Watch Directory**:
A registered stream of Trajectories with a known origin and ownership scope.
_Avoid_: Arbitrary folder

## Learning

**Atom**:
The smallest continuous, same-intent learning unit cut from a Trajectory; it may span multiple chat turns.
_Avoid_: Token, fixed chunk, single turn

**Candidate**:
A reusable pattern supported by an Atom but not yet accepted into a Skill body.
_Avoid_: Staging version

**Weightscore**:
The evidence weight of an Atom-to-Skill assignment; accumulated Weightscore decides when Skill editing is warranted.
_Avoid_: UX Score, probability

**Atom Adoption**:
The durable fact that an Atom contributed evidence to a Skill.
_Avoid_: Skill installation

**Skill**:
A reusable instruction package with an independent version and distribution lifecycle.
_Avoid_: Prompt snippet, Candidate

**Skill Edit**:
An evidence-backed transition that creates or rewrites a Skill from accumulated Candidates.
_Avoid_: Manual text append

## Evolution

**Baby**:
The initial, non-distributable Skill state while its first usable version is assembled.
_Avoid_: Staging

**Main**:
The current stable, distributable Skill version.
_Avoid_: Latest branch

**Staging**:
A proposed update to an existing Main version that is awaiting real-use comparison.
_Avoid_: Candidate, Baby

**Side**:
The exact version lane, Main or Staging, assigned to a session for one Skill.
_Avoid_: Environment

**Session Assignment**:
The stable association from a session to the Skill Side and version it actually used.
_Avoid_: Current installed version

**UX Score**:
A 1–10 assessment of how well an exact Skill version served one Atom.
_Avoid_: Weightscore, generic model score

**Canary**:
A real-use comparison that routes sessions between Main and Staging and selects a winner from version-bound UX evidence.
_Avoid_: Offline evaluation

**Jam**:
A guarded forced-convergence path for an old Staging version whose feedback has plateaued while Candidate evidence continues to accumulate.
_Avoid_: Ordinary promotion

**User Staging**:
An isolated expert-authored client branch that can inform evolution without directly writing Main.
_Avoid_: Staging

## Distribution

**Manifest**:
The immutable, personalized set of Skill identities, Sides, and versions a team client should realize.
_Avoid_: Catalog

**Reconcile**:
The act of aligning local Skill working copies and harness installations with a Manifest while preserving unsent user edits.
_Avoid_: Blind overwrite

**Native Skill**:
A Skill owned by XSkill and participating in its Git, Candidate, and Canary lifecycle.
_Avoid_: SkillHub Skill

**SkillHub Skill**:
A searchable third-party Skill package that does not automatically gain the full Native Skill evolution lifecycle.
_Avoid_: Native Skill
