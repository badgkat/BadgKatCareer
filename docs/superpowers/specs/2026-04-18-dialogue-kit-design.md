# KerbalDialogueKit — ChatterBox Extension Library

A standalone utility library that extends ChatterBox's dialogue rendering with code-triggered scenes, branching, choice capture, and callbacks. Designed for use by any KSP mod that needs character-driven interactive dialogue — not tied to BadgKatCareer or any specific story.

## Vision

ChatterBox's dialogue rendering pipeline (portrait system, line sequencing, audio, instructor models) is the best thing going for kerbal character dialogue. But ChatterBox is scoped to contract-triggered dialogue: linear scenes tied to `CONTRACT_ACCEPTED` / `CONTRACT_SUCCESS` / etc. It cannot be triggered from arbitrary code, cannot branch mid-scene, cannot capture player choices, and cannot call back when a scene ends.

KerbalDialogueKit fills that gap. It wraps ChatterBox's rendering with a higher-level API that any mod can use to:

- Trigger a dialogue scene from code, not just from contract state transitions
- Branch dialogue based on arbitrary game state or flags
- Pause mid-scene and present choice cards, capturing the player's selection
- Receive callbacks when scenes end, choices are made, or dialogue lines complete
- Queue multiple scenes and manage their playback order
- Define disposition-aware line variants (generic flag-based, not tied to specific characters)

The library is deliberately generic. It does not define characters, does not assume a campaign, does not provide content. It provides infrastructure for other mods to build character-driven interactive experiences.

## Dependencies

- ChatterBox >= 1.0.0 (with rendering pipeline made public, or forked — see Implementation Strategy)
- KSP 1.12.x modding libraries

## Implementation Strategy

### Option 1 (preferred): PR to ChatterBox

ChatterBox's rendering classes are currently `internal sealed`. The smallest useful PR to DymndLab/ChatterBox:

- Make `ScenePopupDefinition` public
- Make `CharacterConfig` public
- Make `LineConfig` public
- Make `ScenePopupController.Enqueue(ScenePopupDefinition)` public

This is a non-breaking change — existing ChatterBox consumers (contract-triggered dialogue) continue working unchanged. Other mods gain the ability to construct and enqueue scenes programmatically.

Frame the PR as "enable other mods to trigger ChatterBox scenes programmatically." Benefits the whole KSP modding community.

### Option 2 (fallback): Fork rendering code

ChatterBox is MIT licensed. If the PR is rejected or slow, fork the rendering code — portrait system (`InstructorPortrait`), dialogue sequencing (`ScenePopupController`), audio management — into KerbalDialogueKit directly. Same rendering, but public API.

Both options produce the same KerbalDialogueKit API from a consumer's perspective. Whether the renderer is ChatterBox-with-public-API or our-fork is a swappable implementation detail.

### Option 3 (long-term): Replace ChatterBox entirely

If KerbalDialogueKit proves more capable and ChatterBox-dependent mods migrate, the library could eventually handle contract-triggered dialogue too. Not a goal for the initial release — ChatterBox's contract integration works fine.

---

## API Design

### Core Concepts

**Scene:** A sequence of dialogue lines with one or two speakers, optional branching, optional choices, and optional callbacks. The unit of interactive dialogue.

**Line:** A single piece of dialogue text. Attributes: speaker, text, animation, optional flag-based visibility.

**Choice:** A pause in the scene presenting 2-4 cards to the player. Selecting a card sets a flag and continues the scene on a chosen branch.

**Flag:** A string key with a value, set by the consuming mod or by choices within a scene. Lines and branches can check flags to determine visibility. Flags are scoped per-save.

**Callback:** A delegate the consuming mod provides when enqueueing a scene. Fired on scene end, choice made, line activated, or scene cancelled.

### Scene Definition (Code)

```csharp
var scene = new DialogueScene("duna_approach")
    .WithCharacter("A", CharacterConfig.Wernher())   // convenience factories provided
    .WithCharacter("B", CharacterConfig.Gus())
    .WithTitle("The Approach Debate")
    .AddLine("A", "The spectrographic data is extraordinary.")
    .AddLine("A", "I want a full science package. Heavy lander, deep experiments.",
        animation: "idle_wonder")
    .AddLine("B", "Heavy means expensive. And we've never landed anything this big.",
        animation: "idle_disappointed")
    .AddChoice("approach", new[]
    {
        new DialogueChoice("science_heavy", "Back Wernher: Go big on science"),
        new DialogueChoice("engineering_first", "Back Gus: Prove the engineering"),
        new DialogueChoice("compromise", "Compromise: Do both, smaller")
    })
    .AddLine("A", "This is most excellent!",
        visibleIf: "approach == science_heavy")
    .AddLine("B", "Smart call, boss.",
        visibleIf: "approach == engineering_first")
    .AddLine("A", "A reasonable compromise.",
        visibleIf: "approach == compromise")
    .OnChoiceMade((choiceId, value) => {
        Debug.Log($"Player chose {value} for {choiceId}");
    })
    .OnSceneEnd(() => {
        Debug.Log("Scene complete");
    });

DialogueKit.Enqueue(scene);
```

### Scene Definition (Config)

For mods that want data-driven scenes without C#:

```cfg
DIALOGUE_SCENE
{
    id = duna_approach
    title = The Approach Debate

    CHARACTER
    {
        id = A
        name = Wernher
        model = Instructor_Wernher
        color = #FF82B4E8
    }

    CHARACTER
    {
        id = B
        name = Gus
        model = Strategy_MechanicGuy
        color = #FFFFC078
    }

    LINE
    {
        speaker = A
        text = The spectrographic data is extraordinary.
    }

    LINE
    {
        speaker = A
        animation = idle_wonder
        text = I want a full science package. Heavy lander, deep experiments.
    }

    LINE
    {
        speaker = B
        animation = idle_disappointed
        text = Heavy means expensive. And we've never landed anything this big.
    }

    CHOICE
    {
        id = approach
        OPTION { value = science_heavy; text = Back Wernher: Go big on science }
        OPTION { value = engineering_first; text = Back Gus: Prove the engineering }
        OPTION { value = compromise; text = Compromise: Do both, smaller }
    }

    LINE
    {
        speaker = A
        visibleIf = approach == science_heavy
        text = This is most excellent!
    }

    LINE
    {
        speaker = B
        visibleIf = approach == engineering_first
        text = Smart call, boss.
    }

    LINE
    {
        speaker = A
        visibleIf = approach == compromise
        text = A reasonable compromise.
    }
}
```

Scenes defined in config can be enqueued by ID from code:

```csharp
DialogueKit.EnqueueById("duna_approach",
    onChoiceMade: (id, val) => HandleDunaChoice(val),
    onSceneEnd: () => ShowResults());
```

### Flag System

Flags are string key-value pairs scoped per-save. Consumers set flags externally; lines and choices use them for visibility logic.

```csharp
DialogueKit.Flags.Set("wernherDisposition", "Enthusiastic");
DialogueKit.Flags.Set("completedDunaFlyby", "true");
DialogueKit.Flags.Get("wernherDisposition");  // returns "Enthusiastic"
```

Flag expressions in `visibleIf`:

- `flagName == value`
- `flagName != value`
- `flagName in (value1, value2)`
- `flagName > number` (for numeric flags)
- Combinations with `&&` and `||`

Example line variants by disposition:

```cfg
LINE
{
    speaker = A
    visibleIf = wernherDisposition == Enthusiastic
    text = This is most excellent! I have been waiting for this moment!
}

LINE
{
    speaker = A
    visibleIf = wernherDisposition == Skeptical
    text = Fine. I suppose we can make this work.
    animation = idle_disappointed
}

LINE
{
    speaker = A
    visibleIf = wernherDisposition in (Neutral, Supportive)
    text = An interesting opportunity.
}
```

Exactly one variant fires per line slot — the first one whose `visibleIf` evaluates true. If none match, the line is skipped.

### Choice Capture

When a scene hits a CHOICE node:
1. Dialogue pauses
2. ChoiceOverlay renders cards with option text
3. Player clicks a card
4. The choice flag is set: `Flags.Set(choice.id, option.value)`
5. Scene continues, evaluating `visibleIf` against the new flag
6. `OnChoiceMade(choiceId, value)` callback fires

Options can include a description or effects preview:

```cfg
OPTION
{
    value = science_heavy
    text = Back Wernher: Go big on science
    description = Full science package, +40% Duna biome science. Wernher happy, Gus grumpy.
}
```

The description appears below the option text in the card.

### Callbacks

Scenes can register callbacks at multiple points:

- `OnSceneStart(scene)` — fires when the scene opens
- `OnLineActivated(lineIndex, line)` — fires when a line is clicked to
- `OnChoiceMade(choiceId, value)` — fires when a player selects a choice option
- `OnSceneEnd(scene)` — fires when the scene closes normally
- `OnSceneCancelled(scene)` — fires if the player closes the scene early

Callbacks are optional. Consumer mods register what they care about.

### Queue Management

Multiple scenes can be enqueued. They play in order. Consumers can:

- `DialogueKit.Enqueue(scene)` — add to end of queue
- `DialogueKit.EnqueuePriority(scene)` — add to front of queue
- `DialogueKit.IsPlaying()` — check if a scene is active
- `DialogueKit.QueueLength()` — count pending scenes
- `DialogueKit.Clear()` — drop all pending scenes (use sparingly)

### Audio and Music Hooks

Scenes can specify audio just like ChatterBox:

- `sceneAudio` — background audio clip for the scene
- `lineAudio` per LINE — clip played when the line activates
- `sceneLevel` / `lineLevel` — volume levels on ChatterBox's -10 to +10 scale

Additionally, KerbalDialogueKit provides hooks for music switching integration:

```csharp
scene.WithMusicCue("anomaly_mystery")   // fires a named cue consumers can listen for
scene.OnStart(() => MusicSwitcher.Play("tension"))
scene.OnEnd(() => MusicSwitcher.Resume())
```

Music cue events are published via `GameEvents` so any music mod (Music Switcher, custom, etc.) can subscribe.

---

## Library Structure

Single assembly: `KerbalDialogueKit.dll` in `GameData/KerbalDialogueKit/Plugins/`.

```
KerbalDialogueKit/
├── Core/
│   ├── DialogueKit.cs              // Public static API entry point
│   ├── DialogueScene.cs            // Scene builder + data
│   ├── DialogueLine.cs             // Line definition with visibleIf
│   ├── DialogueChoice.cs           // Choice and option definitions
│   ├── SceneQueue.cs               // Queue management
│   └── FlagSystem.cs               // Per-save flag storage
│
├── Rendering/
│   ├── SceneRenderer.cs            // Bridges to ChatterBox (or fork)
│   ├── ChoiceOverlay.cs            // Mid-scene choice UI
│   ├── LineEvaluator.cs            // visibleIf expression parser
│   └── CharacterConfigFactory.cs   // Convenience factories
│
├── Config/
│   ├── DialogueSceneLoader.cs      // Parse DIALOGUE_SCENE cfg nodes
│   └── FlagExpressionParser.cs     // Parse visibleIf expressions
│
├── Persistence/
│   └── FlagScenario.cs             // ScenarioModule storing per-save flags
│
└── Events/
    └── DialogueEvents.cs           // GameEvents for music hooks, etc.
```

---

## Scope Summary

- **Single library DLL:** `KerbalDialogueKit.dll`
- **Public API surface:** ~10-15 classes
- **Dependencies:** ChatterBox (with public API or forked), KSP 1.12.x
- **Content:** None — this is pure infrastructure
- **Uses:** BadgKatDirector, BadgKatCareer, and any future mod wanting character-driven dialogue
- **License recommendation:** MIT (matches ChatterBox, encourages reuse)
