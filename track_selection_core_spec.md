# Track Selection Sets for MOQ Transport

## Status

This document defines Track Selection Sets for MOQ Transport, including
two selection policies (Top-N and Bandwidth-Aware) and one switching
mechanism (Hard). Additional policies and switching mechanisms may be
defined in extension documents.

---

## 1. Introduction

A subscriber often needs the relay to forward only a subset of tracks
from a namespace. Two common scenarios drive this:

- **Active speaker selection**: In a video conference with 100
  participants, the subscriber wants only the top-5 loudest speakers.
  The set of selected speakers changes over time as people speak.

- **Quality adaptation**: For each selected speaker, the publisher
  offers multiple quality renditions (720p, 1080p, 4K). The relay
  should forward only the rendition that fits the subscriber's
  available bandwidth.

These scenarios require three capabilities that this document defines:

1. **Selection Set Membership**: How tracks are grouped as candidates
   for selection.
2. **Selection Policy**: The algorithm that decides which tracks within
   a set are selected (forwarded) and which are deselected.
3. **Switching Mechanism**: How the transition from selected to
   deselected (and vice versa) is executed at the media delivery level.

### 1.1. Relationship to Existing Work

This specification unifies and replaces the following proposals:

- Top Tracks Filter (PR #1830) -> Policy 0x0 (Top-N)
- SSTS bandwidth allocation (draft-wilaw-moq-moqt-ssts) -> Policy 0x1 (Bandwidth-Aware)
- SWITCH_FROM Hard mode (PR #1674) -> Switch Mode 0x0 (Hard)

The SWITCH_FROM parameter (PR #1674) is unchanged; its Mode field
references the Switch Mode registry defined here.

## 2. Selection Sets

A **Selection Set** is a group of tracks from which a publisher (or
relay) forwards a selected subset to the subscriber. Tracks not in
the selected subset are not forwarded.

### 2.1. Properties of a Selection Set

Each selection set has:

- **Set ID** (varint): Unique identifier within the session.
- **Members**: The tracks currently in the set.
- **Selected Subset**: The members whose objects are being forwarded.
- **Selection Policy**: The algorithm determining the selected subset.
- **Switch Mode**: How transitions are executed when the selected
  subset changes.

A track MUST be a member of at most one selection set at a time.
An endpoint that receives an assignment of a track to a second set
MUST respond with REQUEST_ERROR.

### 2.2. Implicit Membership (Namespace-Scoped)

A selection set spanning all tracks in a namespace is created by
including the TRACK_SELECTION parameter in SUBSCRIBE_TRACKS:

~~~
TRACK_SELECTION {
  Type (vi64) = 0x29,
  Length (vi64),
  Policy ID (vi64),
  Switch Mode (vi64),
  [Policy-specific fields]
}
~~~

All tracks published within the matching namespace are implicitly
members of one selection set. The publisher evaluates the selection
policy to determine which tracks are in the selected subset.

A TRACK_SELECTION with Length=0 removes the selection (all tracks
pass). This may appear in REQUEST_UPDATE.

TRACK_SELECTION MAY appear in SUBSCRIBE_TRACKS or REQUEST_UPDATE
for it.

### 2.3. Explicit Membership (Per-Track Assignment)

A subscriber assigns an individual track to a selection set by
including SELECTION_SET_ASSIGNMENT in PUBLISH_OK or REQUEST_UPDATE:

~~~
SELECTION_SET_ASSIGNMENT {
  Type (vi64) = 0x41,
  Length (vi64),
  Set ID (vi64),
  Policy ID (vi64),
  [Policy-specific fields]
}
~~~

The Set ID identifies the selection set. If the set does not exist,
it is created. Tracks assigned with the same Set ID are members of
the same set.

A SELECTION_SET_ASSIGNMENT with Length=0 removes the track from its
current selection set without reassignment. A new assignment with a
different Set ID moves the track.

A track is removed from its selection set when:
- The subscription ends (UNSUBSCRIBE or PUBLISH_DONE)
- A new SELECTION_SET_ASSIGNMENT with a different Set ID is received
- SELECTION_SET_ASSIGNMENT with Length=0 is received

When a selection set has no remaining members, it is deleted.

## 3. Track State Machine

Every track that is a member of a selection set is in one of three
states:

~~~
        +--------------+
        |   UNKNOWN    |
        |  (no state)  |
        +--------------+
             ^    |
             |    |     Newly Selected
PUBLISH_DONE |    +---> PUBLISH FWD=1 ------> +-----------+
(purge)      |                                | SELECTED  |
             |    +---> NOTIFY FWD=1 -------> |           |
             |    |     Reselected            +-----------+
        +--------------+                           |
        | DESELECTED   | <---- NOTIFY FWD=0  <-----+
        | (state kept) |       Demoted
        +--------------+
~~~

### 3.1. State Transitions

**UNKNOWN -> SELECTED**: The publisher sends PUBLISH with Forward=1.
This occurs when a track first enters the selected subset.

**SELECTED -> DESELECTED**: The publisher sends PUBLISH_STATE_NOTIFY
with Forward=0. The track leaves the selected subset. The switching
mechanism (Section 5) determines when this takes effect.

**DESELECTED -> SELECTED**: The publisher sends PUBLISH_STATE_NOTIFY
with Forward=1. A previously deselected track re-enters the selected
subset. This also updates the Joining Location.

**DESELECTED -> UNKNOWN**: The publisher sends PUBLISH_DONE to purge
state for this track. If the track is later selected again, a new
PUBLISH is required.

### 3.2. Deselected List Management

Publishers SHOULD retain state for recently deselected tracks to
enable efficient reselection via PUBLISH_STATE_NOTIFY (Forward=1)
instead of a new PUBLISH message.

Publishers MAY send PUBLISH_DONE to purge old deselected tracks and
bound memory usage. Relays SHOULD keep the deselected list large
enough to avoid excessive PUBLISH churn for oscillating tracks, but
small enough to protect relay resources.

### 3.3. Universality

This state machine is used identically by all selection policies and
switching mechanisms defined in this document and in extensions.
Implementations need only one state machine implementation regardless
of how many policies they support.

## 4. Selection Policies

A selection policy is an algorithm that determines which members of a
selection set are in the selected subset (Forward=1) and which are
not (Forward=0).

Policies are identified by a Policy ID (varint) and registered in
an IANA registry (Section 10.1). This document defines two policies:

| Policy ID | Name             | Scope      | Description                   |
|----------:|:-----------------|:-----------|:------------------------------|
| 0x0       | Top-N            | Namespace  | Select N highest by property  |
| 0x1       | Bandwidth-Aware  | Per-track  | Select best quality per set   |

An endpoint advertises supported policies in SETUP via the
SELECTION_POLICIES option (Section 6.2). An endpoint MUST NOT send
a TRACK_SELECTION or SELECTION_SET_ASSIGNMENT with a Policy ID not
in the peer's SELECTION_POLICIES list.

### 4.1. Policy 0x0: Top-N Selection

Top-N selects the MaxTracks tracks within a namespace that have the
highest values for a specified Track or Object Property.

#### 4.1.1. Wire Format

~~~
TRACK_SELECTION with Policy 0x0 {
  Type (vi64) = 0x29,
  Length (vi64),
  Policy ID (vi64) = 0x0,
  Switch Mode (vi64),
  Property Type (vi64),
  MaxTracks (vi64),
}
~~~

**Policy ID**: 0x0.

**Switch Mode**: Registered switch mode for transitions (Section 5).

**Property Type**: The Track or Object Property used for ranking.
MUST be even, i.e. a single integer value (see moq-key-value-pair).
Object Property values override Track Property values of the same
type. An endpoint MUST reject a non-even Property Type with
REQUEST_ERROR (INVALID_FILTER).

**MaxTracks**: Maximum number of tracks selected concurrently.
MUST NOT be 0. MUST NOT exceed MAX_SELECTED_TRACKS (Section 6.1).
A violation of either is a PROTOCOL_VIOLATION.

#### 4.1.2. Selection Rules

1. A track is **selected** if it has published an object with one of
   the MaxTracks highest values for Property Type among all tracks in
   the namespace.

2. **Tie-breaking**: When tracks have the same property value, the
   track whose qualifying object was delivered earlier wins. A
   selected track remains selected until another track publishes a
   strictly higher value that demotes it out of the top set.

3. **Timeout**: SELECTION_TIMEOUT (Track Property 0x32, default
   1000 milliseconds) limits how long a track remains selected
   without publishing a new object carrying Property Type. When the
   timeout elapses, the track is deselected.

4. **Out-of-order objects**: Objects received with a location lower
   than the largest received for that track are ignored for selection
   evaluation. However, if the track is already selected, such objects
   still pass the filter.

5. When the TRACK_SELECTION is updated via REQUEST_UPDATE, the
   publisher re-evaluates with the new parameters and signals all
   resulting changes (newly selected, newly deselected, unchanged).

#### 4.1.3. Interaction with Individual SUBSCRIBE

If a track in the namespace is also individually subscribed via
SUBSCRIBE, its state MUST NOT be modified by the Top-N selection.
The individually subscribed track does not count against MaxTracks.
The subscriber receives up to MaxTracks + (individually subscribed)
tracks from that namespace.

#### 4.1.4. Relay Aggregation for Top-N

When multiple downstream subscribers use Policy 0x0 with the same
Property Type on the same namespace:

- The relay aggregates upstream using max(MaxTracks) across all
  downstream subscribers.
- The relay evaluates per-subscriber Top-N locally from the
  aggregated data, forwarding only each subscriber's MaxTracks.

For the well-known property `audio_level`, relays SHOULD precompute
the Top-N ordering as objects arrive, independent of subscriber
connections. New subscribers whose TRACK_SELECTION uses `audio_level`
can be served from precomputed state.

When subscribers use different Property Types on the same namespace,
the relay MUST maintain separate evaluations. It MAY subscribe
upstream without filtering and evaluate all policies locally.

### 4.2. Policy 0x1: Bandwidth-Aware Selection

Bandwidth-Aware selection picks the highest-quality track within each
selection set that fits within the available bandwidth. It is designed
for sets of tracks representing the same content at different bitrates
(e.g., 720p, 1080p, 4K renditions of the same video source).

#### 4.2.1. Wire Format

~~~
SELECTION_SET_ASSIGNMENT with Policy 0x1 {
  Type (vi64) = 0x41,
  Length (vi64),
  Set ID (vi64),
  Policy ID (vi64) = 0x1,
  Throughput Threshold (vi64),
  Set Weight (vi64),
  Activate Switching (vi64),
  Set Rank (8),
}
~~~

**Set ID**: Identifies the selection set. Tracks with the same Set ID
are in the same set.

**Policy ID**: 0x1.

**Throughput Threshold**: Minimum throughput in kbps required to
select this track. The publisher uses this to determine whether a
track fits within the bandwidth allocated to its set.

**Set Weight**: Integer (1-10) establishing proportional bandwidth
distribution among sets with the same rank. Higher weight means more
bandwidth share.

**Activate Switching**: Selection begins once at least this many
tracks are assigned to the set. 0 pauses switching for the set.

**Set Rank**: 8-bit unsigned integer (0-255). Lower values indicate
higher priority. Sets with lower rank receive their full allocation
before sets with higher rank receive any bandwidth.

#### 4.2.2. Bandwidth Allocation Algorithm

The publisher evaluates the selection when a new group arrives on
any track in an active selection set. The algorithm allocates
available bandwidth across sets using strict priority by rank,
then proportional allocation by weight within each rank tier:

~~~
B_remaining = B_total   // total available downstream bandwidth

for each distinct rank value r, in ascending order:
  active_sets = { sets with rank == r AND enough members to activate }
  sum_weight = sum of set.weight for each set in active_sets

  repeat:
    saturated = empty
    for each set in active_sets:
      set.target = B_remaining * set.weight / sum_weight
      if set.target >= lowest threshold in set:
        set.selected = track with highest threshold <= set.target
        if set.selected.threshold == highest threshold in set:
          saturated.add(set)     // set can't use more bandwidth
      else:
        set.selected = null     // no track fits

    // Reallocate unused bandwidth from saturated sets
    for each set in saturated:
      active_sets.remove(set)
      sum_weight -= set.weight
    if saturated is empty:
      break                     // stable allocation reached

  B_remaining -= sum of set.selected.threshold for all sets at rank r
~~~

**Result**: Exactly one track per active set has Forward=1. All other
tracks in the set have Forward=0.

#### 4.2.3. Upstream Forwarding

The publisher SHOULD maintain Forward=1 upstream for ALL tracks in a
selection set, regardless of downstream forwarding state. This ensures
content is available for immediate quality switching when bandwidth
conditions change.

#### 4.2.4. Set Lifecycle

A selection set is created when the first SELECTION_SET_ASSIGNMENT
with its Set ID is received. Selection begins once the number of
assigned tracks meets or exceeds Activate Switching.

A set is deleted when all its members are removed (via UNSUBSCRIBE,
PUBLISH_DONE, or reassignment). Receiving PUBLISH_DONE or UNSUBSCRIBE
for a member decrements the activation count (with a floor of zero).

Set-level properties (Weight, Rank, Activate Switching) are updated
when any member's SELECTION_SET_ASSIGNMENT is updated. The most
recently received values take effect.

#### 4.2.5. Relay Aggregation for Bandwidth-Aware

Selection sets from different downstream subscribers are independent.
No cross-subscriber aggregation is performed.

The relay evaluates the bandwidth allocation algorithm per downstream
subscriber based on that subscriber's available bandwidth and set
configuration.

## 5. Switching Mechanisms

A switching mechanism defines how the transition between selected and
deselected states is executed at the media delivery level.

Switching mechanisms are identified by a Switch Mode (varint) and
registered in an IANA registry (Section 10.2).

This document defines one switching mechanism:

### 5.1. Hard Switch (Mode 0x0)

Hard Switch is the default and MUST be supported by all endpoints
that support selection sets.

**On deselection** (SELECTED -> DESELECTED):

1. The publisher immediately stops forwarding new objects for the
   track.
2. Outstanding streams for the track are reset.
3. Objects already in flight may still arrive at the subscriber.
4. The publisher sends PUBLISH_STATE_NOTIFY with Forward=0.

**On selection** (UNKNOWN -> SELECTED or DESELECTED -> SELECTED):

1. The publisher begins forwarding objects from the joining location.
2. The publisher sends PUBLISH (new) or PUBLISH_STATE_NOTIFY with
   Forward=1 (reselection).

Hard Switch is simple and always correct. It may produce a brief gap
or discontinuity at the switch point. For seamless transitions, use
the Aligned switch mode defined in an extension document.

### 5.2. Switch Mode Registry

| Mode | Name    | Specification   | Requirement             |
|-----:|:--------|:----------------|:------------------------|
| 0x0  | Hard    | This document   | MUST support            |
| 0x1  | Aligned | Extension       | OPTIONAL                |

Additional modes are registered via IANA with Specification Required
policy. An endpoint that receives a Switch Mode it does not support
MUST respond with REQUEST_ERROR.

## 6. Setup Negotiation

### 6.1. MAX_SELECTED_TRACKS (Setup Option 0x09)

~~~
MAX_SELECTED_TRACKS {
  Type (vi64) = 0x09,
  Value (vi64),
}
~~~

Limits the maximum number of tracks the peer may have in the selected
state concurrently, across all selection sets. Default: 0 (selection
sets not supported). If exceeded: PROTOCOL_VIOLATION.

### 6.2. SELECTION_POLICIES (Setup Option 0x0A)

~~~
SELECTION_POLICIES {
  Type (vi64) = 0x0A,
  Count (vi64),
  Policy ID (vi64) ...,
}
~~~

Lists the selection policies the endpoint supports. Default: empty
(selection sets not supported). An endpoint MUST NOT send a
TRACK_SELECTION or SELECTION_SET_ASSIGNMENT with a Policy ID not in
the peer's SELECTION_POLICIES list. Violation: PROTOCOL_VIOLATION.

### 6.3. Capability Summary

An endpoint supports selection sets when it sends MAX_SELECTED_TRACKS
> 0 and a non-empty SELECTION_POLICIES list. Both MUST be present.

## 7. Track Properties

### 7.1. SELECTION_TIMEOUT (Property Type 0x32)

SELECTION_TIMEOUT is a Track Property. Its value is a varint
representing milliseconds. Default: 1000.

It limits how long a track remains in the selected state without
publishing a new object carrying the selection property (Property Type
from the TRACK_SELECTION). When the timeout elapses before a new
qualifying object arrives, the track is deselected.

## 8. Relay Behavior

### 8.1. Processing Pipeline

Relay behavior follows the same pipeline regardless of policy:

1. **Receive**: Objects arrive from upstream publishers.
2. **Extract**: Read property values from object metadata.
3. **Evaluate**: Run the selection policy for each affected set.
4. **Diff**: Determine which tracks changed state.
5. **Switch + Signal**: Apply the switch mode and send PUBLISH,
   PUBLISH_STATE_NOTIFY, or PUBLISH_DONE as appropriate.

Steps 1-3 are policy-independent in structure (parameterized by
policy in step 3). Steps 4-5 are policy-independent entirely.

### 8.2. Precomputation

For well-known properties (e.g., audio_level), relays SHOULD
continuously run steps 1-3 as objects arrive, independent of any
subscriber. When a subscriber connects with a matching TRACK_SELECTION,
the relay can serve it from precomputed state without delay.

### 8.3. Upstream Subscription

For namespace-scoped selection sets (Policy 0x0), the relay MUST
subscribe to all tracks in the namespace upstream in order to
evaluate the selection policy. The relay SHOULD aggregate across
downstream subscribers to minimize upstream subscriptions (see
policy-specific aggregation rules).

For per-track selection sets (Policy 0x1), the relay SHOULD maintain
Forward=1 upstream for all tracks in a selection set, regardless of
downstream forwarding state.

## 9. Client Behavior

### 9.1. Subscriber

1. **SETUP**: Send MAX_SELECTED_TRACKS and SELECTION_POLICIES.

2. **Namespace-scoped selection**: Send SUBSCRIBE_TRACKS with
   TRACK_SELECTION containing Policy ID, Switch Mode, and
   policy-specific fields.

3. **Per-track selection**: On receiving PUBLISH for a track,
   respond with PUBLISH_OK containing SELECTION_SET_ASSIGNMENT
   to assign it to a selection set.

4. **Handle state changes**:
   - PUBLISH FWD=1: New track selected. Render/consume it.
   - NOTIFY FWD=0: Track deselected. Stop rendering after
     switch completes per the switch mode.
   - NOTIFY FWD=1: Track reselected. Resume rendering.

5. **Update**: Send REQUEST_UPDATE with updated TRACK_SELECTION or
   SELECTION_SET_ASSIGNMENT. Length=0 to remove.

### 9.2. Publisher / Relay-as-Publisher

1. **On SUBSCRIBE_TRACKS with TRACK_SELECTION**: Validate Policy ID
   against SELECTION_POLICIES. Validate MaxTracks against
   MAX_SELECTED_TRACKS. Evaluate policy and send PUBLISH for each
   initially selected track.

2. **Continuously re-evaluate** the selection policy as new objects
   arrive and properties change.

3. **On selection change**: Apply the switch mode and send the
   appropriate signaling messages (Section 3.1).

4. **Manage deselected list**: Retain recently deselected tracks
   for efficient reselection. Purge old entries via PUBLISH_DONE.

## 10. IANA Considerations

### 10.1. Selection Policies Registry

This document establishes the "MOQT Selection Policies" registry.
Registration policy: Specification Required.

| Policy ID | Name            | Scope     | Specification   |
|----------:|:----------------|:----------|:----------------|
| 0x0       | Top-N           | Namespace | Section 4.1     |
| 0x1       | Bandwidth-Aware | Per-track | Section 4.2     |

### 10.2. Switch Modes Registry

This document establishes the "MOQT Switch Modes" registry.
Registration policy: Specification Required.

| Mode | Name | Specification   |
|-----:|:-----|:----------------|
| 0x0  | Hard | Section 5.1     |

### 10.3. Setup Options

| Type | Name               | Specification   |
|-----:|:-------------------|:----------------|
| 0x09 | MAX_SELECTED_TRACKS | Section 6.1    |
| 0x0A | SELECTION_POLICIES  | Section 6.2    |

### 10.4. Parameters

| Type | Name                     | Specification   |
|-----:|:-------------------------|:----------------|
| 0x29 | TRACK_SELECTION          | Section 2.2     |
| 0x41 | SELECTION_SET_ASSIGNMENT | Section 2.3     |

### 10.5. Properties

| Type | Name              | Scope | Specification   |
|-----:|:------------------|:------|:----------------|
| 0x32 | SELECTION_TIMEOUT | Track | Section 7.1     |

---

## Appendix A: Composition — Conferencing Example

This appendix illustrates how Policy 0x0 (Top-N) and Policy 0x1
(Bandwidth-Aware) compose to solve the conferencing use case.

### A.1. Setup

~~~
Client -> Relay (SETUP):
  MAX_SELECTED_TRACKS: 9
  SELECTION_POLICIES: [0x0, 0x1]

Relay -> Client (SETUP):
  MAX_SELECTED_TRACKS: 25
  SELECTION_POLICIES: [0x0, 0x1]
~~~

### A.2. Layer 1 — Speaker Selection (Top-N)

The subscriber creates an implicit selection set across the meeting
namespace using Top-N:

~~~
Client -> Relay:
  SUBSCRIBE_TRACKS {
    namespace: (mocha_v1, acme, org1, media, team1, ch1, mtg42)
    TRACK_SELECTION {
      Policy: 0x0 (Top-N)
      Switch Mode: 0x0 (Hard)
      Property: audio_level
      MaxTracks: 5
    }
  }
~~~

The relay subscribes upstream, evaluates audio_level across all
tracks, and sends PUBLISH (FWD=1) for the top-5 speakers' tracks.

### A.3. Layer 2 — Quality Selection (Bandwidth-Aware)

For each PUBLISH received for a video rendition track, the subscriber
assigns it to a per-participant selection set:

~~~
Client -> Relay:
  PUBLISH_OK {
    track: video-alice-720p
    SELECTION_SET_ASSIGNMENT {
      Set ID: 1
      Policy: 0x1 (Bandwidth-Aware)
      Throughput Threshold: 500    // kbps
      Set Weight: 3
      Activate Switching: 2
      Set Rank: 0
    }
  }

  PUBLISH_OK {
    track: video-alice-1080p
    SELECTION_SET_ASSIGNMENT {
      Set ID: 1
      Policy: 0x1 (Bandwidth-Aware)
      Throughput Threshold: 1500
      Set Weight: 3
      Activate Switching: 2
      Set Rank: 0
    }
  }
~~~

The relay evaluates bandwidth allocation and selects the best
rendition per participant.

### A.4. Independence of Layers

- **Speaker change** (Layer 1): When frank's audio_level exceeds eve's,
  the relay deselects eve's tracks (NOTIFY FWD=0) and selects frank's
  tracks (PUBLISH FWD=1). Layer 2 sets for other speakers are
  unaffected.

- **Bandwidth change** (Layer 2): When available bandwidth drops, the
  relay downgrades quality within each per-participant set (e.g.,
  1080p -> 720p). Layer 1 speaker selection is unaffected.

The two layers use the same state machine, same signaling, and can
use the same or different switch modes.

---

## Appendix B: Extension Points

### B.1. New Selection Policies

New policies are registered in the Selection Policies registry.
Each policy:

- Defines policy-specific fields appended to TRACK_SELECTION or
  SELECTION_SET_ASSIGNMENT after the Policy ID.
- Specifies its selection rules.
- Specifies its relay aggregation behavior.
- Uses the existing state machine (Section 3) and signaling.
- Uses registered switch modes (Section 5).

Anticipated future policies:

| Policy ID | Name                 | Description                              |
|----------:|:---------------------|:-----------------------------------------|
| 0x2       | Threshold            | Select all tracks above a property value |
| 0x3       | Bottom-N             | Select N tracks with lowest values       |
| 0x4       | Conditional Assignment | Auto-assign tracks to sets based on conditions |

### B.2. New Switch Modes

New modes are registered in the Switch Modes registry. Each mode:

- Defines the deselection behavior (when and how forwarding stops).
- Defines any selection behavior changes.
- Uses the existing state machine and signaling.

Anticipated future modes:

| Mode | Name      | Description                                      |
|-----:|:----------|:-------------------------------------------------|
| 0x1  | Aligned   | Drain to group boundary before deselecting       |
| 0x2  | Crossfade | Overlap old and new track for blended transition |

### B.3. Relationship to SWITCH_FROM

SWITCH_FROM (PR #1674) allows explicit subscriber-initiated track
switching. Its Mode field references the Switch Modes registry
defined here. When SWITCH_FROM uses Mode 0x0 (Hard), the behavior
is identical to this document's Hard Switch. Future SWITCH_FROM
modes (e.g., Aligned) use the same registry entries.

SWITCH_FROM operates on individual subscriptions rather than
selection sets. It is complementary: selection sets handle
algorithm-driven selection, while SWITCH_FROM handles explicit
subscriber-driven switching.
