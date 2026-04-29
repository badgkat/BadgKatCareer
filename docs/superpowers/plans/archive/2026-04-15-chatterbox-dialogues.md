# ChatterBox Dialogue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add character-driven ChatterBox dialogue scenes to ~40 BadgKatCareer contracts, introducing six kerbal characters who teach spaceflight concepts through goofy personality.

**Architecture:** Each contract gets 1-2 ChatterBox BEHAVIOUR nodes added inside the existing CONTRACT_TYPE block, before the closing brace. Tier 1 contracts (landmarks) get cinematic two-speaker scenes on accept AND complete. Tier 2 contracts get a single quick exchange. Existing stock DialogBox behaviours in AnomalySurveyor files are replaced with ChatterBox equivalents.

**Tech Stack:** ContractConfigurator cfg syntax, ChatterBox v1.0.0 BEHAVIOUR nodes

**Reference:** Full spec at `docs/superpowers/specs/2026-04-15-chatterbox-dialogue-design.md`

---

## Writing Guidelines (for every task)

All dialogue must follow these rules. Read the full character voice guide in the spec before writing any lines.

- **Gene:** Dry, understated, fatherly. "Not that I'm worried..." / "perfectly routine" / "unscheduled disassembly". Green `#FF8BE08A`.
- **Wernher:** Germanic-formal chaos scientist. "This is most promising" / treats explosions as data / drops real space history. Blue `#FF82B4E8`.
- **Gus:** Blue-collar mechanic. "She pulls to the left" / duct tape engineering / uses she/her for vehicles. Orange `#FFFFC078`.
- **Mortimer:** Nervous finance. "Do you have ANY idea how much that costs?" / treats fuel as liquid money. Yellow `#FFFFE066`.
- **Walt:** Smarmy PR. "The public is going to LOVE this" / drafts headlines in real time. Purple `#FFC8A0E8`.
- **Linus:** Nervous junior scientist. "Is that... supposed to look like that?" / audience surrogate. Teal `#FF6ED4C8`.

**Lines:** 1-2 sentences each. Real science in kerbal context. Humor from character, not jokes. No fourth wall. Kerbosene, Koxide, Snacks. No "press X" UI instructions — teach concepts like periapsis, center of lift, orbital mechanics.

**Instructor models:** Gene=`Instructor_Gene`, Wernher=`Instructor_Wernher`, Gus=`Strategy_MechanicGuy`, Mortimer=`Strategy_Mortimer`, Walt=`Strategy_PRGuy`, Linus=`Strategy_ScienceGuy`

**Animations:** `idle` (talking), `true_nodA` (agreeing), `true_smileA` (pleased), `true_thumbsUp` (big win), `idle_wonder` (mystery), `idle_disappointed` (went wrong), `idle_sadA` (concern)

---

## Task 1: FirstScience — Program Opens (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/FirstSteps.cfg`

This is the first dialogue the player ever sees. Introduces Gene and Wernher. Teaches what science collection is.

- [ ] **Step 1: Add accept scene to BKEX_FirstScience**

Insert before the final `}` of the `BKEX_FirstScience` CONTRACT_TYPE block (before line 39):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Welcome to the Space Program

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = Welcome to the Kerbal Space Program. I'm Gene Kerman, and I run Mission Control. That means I worry about everything so you don't have to. }
		LINE { speaker = B; text = And I am Wernher von Kerman, head of the Science Division. I will be providing the experiments, the hypotheses, and occasionally the explosions. The line between those is thinner than you would expect. }
		LINE { speaker = A; text = Here is the deal — we need data. Any data. Fly a rocket, drive a rover, roll a plane down the runway... just collect some science and bring it back. }
		LINE { speaker = B; text = Even a simple temperature reading at the launchpad tells us something we did not know before. That is how science works — one measurement at a time. The universe is under no obligation to make sense, but it does have to hold still while we take notes. }
		LINE { speaker = A; text = Good team, good equipment, good odds. Try to bring back more pieces than you left with. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_FirstScience**

Insert after the accept BEHAVIOUR, still inside the CONTRACT_TYPE:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = First Data

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_nodA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_smileA
		characterBColor = #FF82B4E8

		LINE { speaker = B; text = Data! Actual data! It is not much, but the first measurement is always the most important. Every discovery in history started with someone writing down a number. }
		LINE { speaker = A; text = The space program is officially in business. Not bad for day one. }
		LINE { speaker = B; text = Now — I have been looking at the sky, and I have some ideas about what we should measure next. Several ideas, actually. Gene, how do you feel about rockets? }
		LINE { speaker = A; text = ...I feel like you are about to make my job a lot more interesting. }
	}
```

- [ ] **Step 3: Verify syntax**

Check that the file has matching braces and no syntax errors. The CONTRACT_TYPE for BKEX_FirstScience should now contain: PARAMETER, REQUIREMENT, BEHAVIOUR (accept), BEHAVIOUR (complete), then closing `}`.

---

## Task 2: FirstRover — Meet Gus (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/FirstRover.cfg`

Introduces Gus. Teaches surface exploration. He built the rover and talks about her like a stubborn pet.

- [ ] **Step 1: Add accept scene to BKML_FirstRover**

Insert before the final `}` of the CONTRACT_TYPE block (before line 65):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Meet the Rover

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Gus
		characterBModel = Strategy_MechanicGuy
		characterBAnimation = idle
		characterBColor = #FFFFC078

		LINE { speaker = A; text = Our head engineer has been building something out in the workshop. Gus, you want to show them? }
		LINE { speaker = B; text = She is not pretty, but she runs. I bolted some wheels to a probe core, wired up the batteries, and gave her a good kick. She pulls a little to the left, but she will get you there. }
		LINE { speaker = A; text = "There" being anywhere on the surface you can reach without a rocket. Rovers open up a whole planet for exploration. }
		LINE { speaker = B; text = Just keep her under five meters per second on the hills. She is top-heavy and I am not building another one this week. }
	}
```

- [ ] **Step 2: Add complete scene to BKML_FirstRover**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = She Rolls

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gus
		characterAModel = Strategy_MechanicGuy
		characterAAnimation = true_smileA
		characterAColor = #FFFFC078

		LINE { speaker = A; text = She held together. I was not sure about that left wheel, but she held together. That is all you can ask. }
		LINE { speaker = A; text = Surface mobility changes everything. You are not stuck where you land anymore — the whole ground is yours. I will start working on the next model. Bigger wheels, maybe some headlights. }
	}
```

- [ ] **Step 3: Verify syntax**

---

## Task 3: FirstFlight — Wright Brothers (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/FirstFlight.cfg`

The aviation milestone. Wright brothers parallel. Teaches lift and center of mass/lift relationship.

- [ ] **Step 1: Add accept scene to BKML_FirstFlight**

Insert before the final `}` of the CONTRACT_TYPE (before line 65):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Dream of Flight

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = In 1903, two bicycle mechanics built a machine out of wood and fabric that flew for twelve seconds. Twelve seconds! That was enough to change everything. }
		LINE { speaker = B; text = We have slightly better materials. And a runway that is longer than a sand dune. }
		LINE { speaker = A; text = The physics is simple and beautiful — air moves faster over a curved wing, creating lower pressure on top. That difference in pressure is lift. Fight gravity with geometry. }
		LINE { speaker = A; text = One critical detail — the center of lift should sit just behind the center of mass. That way, if the nose dips, the lift pushes it back up. If it is the other way around... well. You will find out quickly. }
		LINE { speaker = B; text = I have the fire crew standing by. Not that I am worried. Perfectly routine first flight. }
	}
```

- [ ] **Step 2: Add complete scene to BKML_FirstFlight**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Aviation Arrives

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_smileA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_nodA
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = Flight confirmed. The ground crew is cheering, the engineers are already arguing about the next design, and someone just ordered a second runway. }
		LINE { speaker = B; text = Wings beat gravity today. Next we see how fast they can go. Those bicycle mechanics would be proud. }
	}
```

- [ ] **Step 3: Verify syntax**

---

## Task 4: Kerbin Monolith — Meet Linus (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg`

Introduces Linus. Replaces existing stock DialogBox. The beginning of the anomaly mystery chain.

- [ ] **Step 1: Remove existing DialogBox BEHAVIOUR**

Delete the entire `BEHAVIOUR { type = DialogBox ... }` block (lines 58-98 of the current file).

- [ ] **Step 2: Add accept scene**

Insert a ChatterBox accept BEHAVIOUR in its place:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Strange Readings

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Linus
		characterBModel = Strategy_ScienceGuy
		characterBAnimation = idle
		characterBColor = #FF6ED4C8

		LINE { speaker = B; text = Gene, I need to report something. My gyroscopes have been drifting for weeks. I thought the bearings were wearing out, but Wernher checked — it is not mechanical. Something is generating a magnetic field north of the KSC. }
		LINE { speaker = A; text = A magnetic field? How big are we talking? }
		LINE { speaker = B; text = Big enough to affect instruments from here. I have been calling it the Tycho Magnetic Anomaly. Whatever is causing it... it is not natural. At least, not any natural thing I have seen before. }
		LINE { speaker = A; text = That is right in our backyard. Take a rover out there and see what you find. And Linus — keep an open mind. }
		LINE { speaker = B; text = That is what worries me, Gene. I have a very open mind and it is filling up with very strange possibilities. }
	}
```

- [ ] **Step 3: Add complete scene**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The Monolith

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle_wonder
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = Gene... it is a monolith. A perfectly smooth, perfectly black rectangle, just standing there in the grass. The magnetic readings are off the charts. }
		LINE { speaker = B; text = Right in our backyard and we never saw it. Maybe if we had windows in the administration building. }
		LINE { speaker = A; text = I am picking up something else. A signal — very faint, very structured. It is pointing... away. Toward space. Like it wants us to follow it. }
		LINE { speaker = B; text = Someone put this here, Linus. Someone who is not us. }
	}
```

- [ ] **Step 4: Keep existing WaypointGenerator BEHAVIOUR**

The WaypointGenerator BEHAVIOUR (waypoint at lat 0.10, lon -74.57) stays exactly as-is. Only the DialogBox is replaced.

- [ ] **Step 5: Verify syntax**

---

## Task 5: UnmannedSuborbital — Meet Mortimer (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/FirstSteps.cfg`

Introduces Mortimer via budget panic. Teaches what suborbital means and why probes go first. Note: this contract shares a file with FirstScience (Task 1).

- [ ] **Step 1: Add accept scene to BKEX_UnmannedSuborbital**

Insert before the final `}` of the BKEX_UnmannedSuborbital CONTRACT_TYPE block (before line 92):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Going Up

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Mortimer
		characterBModel = Strategy_Mortimer
		characterBAnimation = idle
		characterBColor = #FFFFE066

		LINE { speaker = A; text = The data from our first experiments is clear — there is more to learn above the atmosphere. It is time to go up. Way up. }
		LINE { speaker = B; text = Do you have ANY idea how much Kerbosene costs per kilogram? We are about to set a very large pile of money on fire. Literally. }
		LINE { speaker = A; text = Mortimer Kerman, everybody. Our finance director. He keeps the lights on. }
		LINE { speaker = B; text = Someone has to. Do we at least have to send a kerbal up there? }
		LINE { speaker = A; text = That is exactly the point — we do not. A probe goes first. Cheaper, no Snacks required, and nobody's family calls Mission Control if it does not come back. Suborbital means it goes up, it comes back down. The space part is the bit above 70 kilometers where the sky goes black. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_UnmannedSuborbital**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = We Touched Space

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_thumbsUp
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_smileA
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = Space! Our probe touched actual space and came back in one piece! }
		LINE { speaker = B; text = The first objects to reach space from Earth were not much bigger than this. A V-2 rocket crossed the line in 1944 — it went up, it came down, and everything we knew about the sky changed. Today, we did the same. }
		LINE { speaker = A; text = But going up and coming back down is only the beginning. The real trick is staying up there. }
		LINE { speaker = B; text = Orbit. Yes. That is next. And it is a very different problem — a beautiful, elegant, sideways problem. }
	}
```

- [ ] **Step 3: Verify syntax**

Both CONTRACT_TYPEs in FirstSteps.cfg should now have their BEHAVIOUR nodes.

---

## Task 6: CrewedUpperAtmo — Meet Walt (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/EarlyCrewed.cfg`

Introduces Walt. First crewed mission. This file has two CONTRACT_TYPEs — add to BKEX_CrewedUpperAtmo only in this task.

- [ ] **Step 1: Add accept scene to BKEX_CrewedUpperAtmo**

Insert before the final `}` of the BKEX_CrewedUpperAtmo block (before line 58):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = First Crew

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Walt
		characterBModel = Strategy_PRGuy
		characterBAnimation = idle
		characterBColor = #FFC8A0E8

		LINE { speaker = A; text = Up until now, we have been sending machines. Today we send a kerbal. Not to space — not yet — but high enough that the sky turns dark and the horizon curves. Eighteen thousand meters. }
		LINE { speaker = B; text = Gene! Walt Kerman, Public Relations. The media is going to LOVE this. I am already drafting the headline — "Kerbal Touches the Edge of Forever." No wait — "One Small Step Into the Very High Air." I will workshop it. }
		LINE { speaker = A; text = ...this is Walt. He handles the press. }
		LINE { speaker = B; text = The public wants heroes, Gene. Give me a kerbal in a capsule with a view and I will give you front page coverage for a week. }
		LINE { speaker = A; text = Let us focus on getting them back in one piece first. That is the part the press really cares about. }
	}
```

- [ ] **Step 2: Verify syntax**

Note: CrewedUpperAtmo gets accept only (Tier 1 lite — the complete is less dramatic since CrewedSuborbital is the bigger beat).

---

## Task 7: UnmannedOrbit — Newton's Cannonball (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/UnmannedOrbit.cfg`

THE orbit teaching moment. Newton's cannonball thought experiment. This is the biggest conceptual lesson in the career.

- [ ] **Step 1: Add accept scene to BKEX_UnmannedOrbit**

Insert before the final `}` (before line 57):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Sideways Problem

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = Suborbital is going up and coming down. Orbit is something entirely different. Orbit is the trick of going sideways so fast that you keep falling but never hit the ground. }
		LINE { speaker = B; text = That sounds like missing the ground on purpose. }
		LINE { speaker = A; text = Exactly! Newton imagined a cannon on a very tall mountain. Fire it gently, the ball lands nearby. Fire it harder, it lands farther away. Fire it hard enough — and the ball falls AROUND the planet. The ground curves away as fast as the ball falls toward it. }
		LINE { speaker = A; text = Your periapsis is the lowest point of the orbit — the closest you get to the ground. Your apoapsis is the highest. To circularize, you burn at apoapsis until your periapsis comes up to match. }
		LINE { speaker = B; text = In simple terms — point up, go fast, then turn sideways and go faster. }
		LINE { speaker = A; text = ...that is reductive, but technically not wrong. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_UnmannedOrbit**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Orbit Achieved

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = true_thumbsUp
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = true_smileA
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = It is up there. It will BE up there long after we are gone. That is what orbit means — you gave it enough energy to fall forever and never land. }
		LINE { speaker = B; text = Tracking stations confirm stable orbit. Every pass brings new data. This changes everything — relays, stations, interplanetary transfers... it all starts here. }
		LINE { speaker = A; text = Sputnik was about this size, you know. It beeped. Ours also beeps. We are on track. }
	}
```

- [ ] **Step 3: Verify syntax**

---

## Task 8: CrewedSuborbital — First Kerbal in Space (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/EarlyCrewed.cfg`

The big emotional beat. Gene drops the deadpan. Same file as Task 6 — add to BKEX_CrewedSuborbital.

- [ ] **Step 1: Add accept scene to BKEX_CrewedSuborbital**

Insert before the final `}` of BKEX_CrewedSuborbital (before line 127):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Big One

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle_sadA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = This is the one we have been building toward. A kerbal. In space. Not a probe, not an instrument — a person, sitting on top of a rocket, trusting us to bring them home. }
		LINE { speaker = B; text = In 1961, Yuri Gagarin rode a modified missile to orbit. When asked if he was scared, he said "Should I be?" He was a test pilot. So are ours. }
		LINE { speaker = A; text = Suborbital first. Up past the atmosphere, a few minutes of weightlessness, then back down. The hard part is not getting up there. The hard part is the coming home part. }
		LINE { speaker = B; text = The heat shield will handle reentry. Probably. I ran the numbers twice. }
		LINE { speaker = A; text = ...not helping, Wernher. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_CrewedSuborbital**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = A Kerbal Has Been to Space

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_thumbsUp
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_thumbsUp
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = They are home. They are safe. A kerbal has been to space and come back to tell about it. }
		LINE { speaker = B; text = Today a kerbal saw the black sky and the curved horizon. They know something now that the rest of us can only imagine. }
		LINE { speaker = A; text = I am not going to pretend that was easy for me. But it was worth it. Every bit of it was worth it. }
	}
```

- [ ] **Step 3: Verify syntax**

---

## Task 9: CrewedHomeOrbit (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/CrewedExploration.cfg`

Add to BKEX_CrewedHomeOrbit only. Revisits orbit concept with crew stakes.

- [ ] **Step 1: Add accept scene to BKEX_CrewedHomeOrbit**

Insert before the final `}` of BKEX_CrewedHomeOrbit (before line 289):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = All the Way Around

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = Suborbital was up and back down. This time, we go all the way around. A crew, in orbit, falling around the planet. }
		LINE { speaker = B; text = Remember the difference — suborbital is a lob. Orbit is a circle. The crew will be weightless not for minutes but for as long as they stay up there. They will see sunrise every ninety minutes. }
		LINE { speaker = A; text = And we need them back. Orbit means they have to slow down enough to drop back into the atmosphere. The deorbit burn is just as important as the one that got them up there. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_CrewedHomeOrbit**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Orbit Complete

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_smileA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_nodA
		characterBColor = #FF82B4E8

		LINE { speaker = A; text = One full orbit. The crew saw the whole world from above — every ocean, every mountain, every cloud. }
		LINE { speaker = B; text = And they came home. That is the part that makes this real. We can put kerbals in space AND bring them back. The rest of the solar system just became possible. }
	}
```

- [ ] **Step 3: Verify syntax**

Note: Only add BEHAVIOURs to BKEX_CrewedHomeOrbit, NOT to BKEX_CrewedOrbit or BKEX_CrewedLanding (those are Tier 2).

---

## Task 10: ProbeFlyby — Leaving Home (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/ProbeExploration.cfg`

First time leaving the home system. Only add to BKEX_ProbeFlyby (the first contract in the file).

- [ ] **Step 1: Add accept scene to BKEX_ProbeFlyby**

Insert before the final `}` of BKEX_ProbeFlyby (before line 240):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Beyond Home

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Linus
		characterBModel = Strategy_ScienceGuy
		characterBAnimation = idle
		characterBColor = #FF6ED4C8

		LINE { speaker = A; text = We have orbited our homeworld. We have proven that what goes up can stay up. Now we aim for somewhere else entirely. A flyby — our probe will pass close, gather data, and keep going. }
		LINE { speaker = B; text = A flyby means we do not have to slow down when we get there, right? We just... sail past? }
		LINE { speaker = A; text = Precisely. Stopping takes fuel — sometimes more fuel than getting there. A flyby trades time for efficiency. The probe arrives fast, collects data during the closest approach, and the gravity of the target bends our path onward. }
		LINE { speaker = B; text = Gravity can bend a path? }
		LINE { speaker = A; text = Everything with mass bends the space around it. A planet is a curve in the road. Use it well and you get a free turn. The Voyager probes used this trick to visit four planets with one launch. We can do the same. }
	}
```

- [ ] **Step 2: Add complete scene to BKEX_ProbeFlyby**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = First Contact

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = The data is coming in. Another world, up close, for the first time. I keep staring at the images. It is real. It is all real and it is right there. }
	}
```

- [ ] **Step 3: Verify syntax**

Note: Only add to BKEX_ProbeFlyby. BKEX_ProbeOrbit and BKEX_ProbeLanding are Tier 2 (Task 14).

---

## Task 11: SoundBarrier — Yeager (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/SoundBarrier.cfg`

Complete scene only (no accept — the existing description handles the setup). Yeager parallel. Teaches Mach numbers.

- [ ] **Step 1: Add complete scene to BKML_SoundBarrier**

Insert before the final `}` (before line 72):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Mach 1

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = true_smileA
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = true_nodA
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = In 1947, a pilot named Chuck Yeager flew faster than sound for the first time. They said the airplane would shake apart at the barrier. It did not. Yours did not either. }
		LINE { speaker = B; text = The windows at KSC rattled. I think Mortimer's coffee mug fell off his desk. }
		LINE { speaker = A; text = Mach 1 — 343 meters per second at sea level. "Mach" is simply your speed divided by the speed of sound. Past Mach 1, you outrun your own noise. The sonic boom you heard is the sound catching up all at once. }
	}
```

- [ ] **Step 2: Verify syntax**

---

## Task 12: SuborbitalSpaceplane — SSTO (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/SuborbitalSpaceplane.cfg`

Wernher and Gus. Teaches what makes SSTOs special.

- [ ] **Step 1: Add accept scene to BKML_SuborbitalSpaceplane**

Insert before the final `}` (before line 74):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Runway to Space

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Gus
		characterBModel = Strategy_MechanicGuy
		characterBAnimation = idle
		characterBColor = #FFFFC078

		LINE { speaker = A; text = A spaceplane. One vehicle that takes off from a runway, flies to space, and lands back on the same runway. No stages dropped in the ocean, no parachutes, no recovery boats. Everything comes home. }
		LINE { speaker = B; text = You are describing an airplane that is also a rocket. Those are two things that do not like being the same thing. }
		LINE { speaker = A; text = The air-breathing engines do the heavy lifting in the atmosphere — they are efficient because they use the air itself as oxidizer. Then at high altitude where the air thins out, you switch to rockets for the final push to space. Two engine types, one vehicle. }
		LINE { speaker = B; text = She is going to need a lot of thermal protection. Things get hot past Mach 5. I will reinforce the leading edges. }
	}
```

- [ ] **Step 2: Verify syntax**

Complete scene omitted (Tier 1 lite — the accept carries the teaching, the completedMessage text is sufficient).

---

## Task 13: Kcalbeloh Chain — Wormhole and Black Hole (Tier 1)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/Kcalbeloh_Wormhole.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/Kcalbeloh_BlackHole.cfg`

The narrative climax. Wormhole on accept, Black Hole on complete.

- [ ] **Step 1: Add accept scene to AS_KC_Wormhole**

Insert before the final `}` of the CONTRACT_TYPE (before line 62):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = A Hole in the Sky

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		characterBName = Linus
		characterBModel = Strategy_ScienceGuy
		characterBAnimation = idle
		characterBColor = #FF6ED4C8

		LINE { speaker = A; text = The monolith signals — all of them, from Kerbin to Plock — they were pointing here. To this region of space. And now our instruments confirm what I hardly dared to hypothesize. }
		LINE { speaker = B; text = Wernher... what am I looking at? The gravitational readings do not make any sense. }
		LINE { speaker = A; text = An Einstein-Rosen bridge. In simple terms — a tunnel through spacetime. Two points in the universe connected by a shortcut through the fabric of space itself. The math has predicted them for centuries. Nobody expected to find one. }
		LINE { speaker = B; text = A wormhole. An actual wormhole. Is it... stable? }
		LINE { speaker = A; text = The readings suggest yes. Stable and traversable. Something — or someone — built this. The same intelligence that left the monoliths. They were not just markers. They were a road map. And the road leads through that hole in the sky. }
		LINE { speaker = B; text = Wernher, I have to ask — how concerned should I be? }
		LINE { speaker = A; text = Profoundly. But also exhilarated. Send the probe through. Carefully. }
	}
```

- [ ] **Step 2: Read Kcalbeloh_BlackHole.cfg to find insertion point**

Read the file to determine the correct insertion line.

- [ ] **Step 3: Add complete scene to AS_KC_BlackHole**

Insert a `CONTRACT_SUCCESS` ChatterBox BEHAVIOUR before the final `}` of the BlackHole CONTRACT_TYPE:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The End of the Road

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle_wonder
		characterBColor = #FF8BE08A

		LINE { speaker = A; text = A black hole. An object so massive that space and time collapse around it. Light goes in and does not come out. The boundary — the event horizon — is not a surface. It is a point of no return. }
		LINE { speaker = B; text = The monoliths on Kerbin, on the Mun, on Plock... they were all pointing here. To this. }
		LINE { speaker = A; text = Whoever built them wanted us to see this. To know it was here. A black hole is not just destruction — it is the most extreme laboratory in the universe. Physics behaves differently here. Time itself runs slower near the horizon. }
		LINE { speaker = B; text = Wernher... what do we do now? }
		LINE { speaker = A; text = We study it. We learn from it. And we remember that the universe is much, much larger than we imagined. The monoliths were an invitation. We accepted. }
	}
```

- [ ] **Step 4: Verify syntax for both files**

---

## Task 14: Tier 2 — Exploration Contracts

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/ProbeExploration.cfg` (ProbeOrbit, ProbeLanding)
- Modify: `GameData/BadgKatCareer/ContractPacks/Exploration/CrewedExploration.cfg` (CrewedOrbit, CrewedLanding, CrewedFlyby if it exists)
- Modify: `GameData/BadgKatCareer/ContractPacks/MissionControl/FirstRelay.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/MissionControl/RelayConstellation.cfg`

Quick 1-2 line exchanges. One BEHAVIOUR per contract.

- [ ] **Step 1: Add to BKEX_ProbeOrbit (complete)**

Insert before the final `}` of BKEX_ProbeOrbit:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Orbital Insertion

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = true_nodA
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = Stable orbit confirmed. Our probe is circling a new world, sending data with every pass. Each orbit teaches us something the last one missed. }
	}
```

- [ ] **Step 2: Add to BKEX_ProbeLanding (complete)**

Insert before the final `}` of BKEX_ProbeLanding:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Touchdown

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = Surface contact confirmed. The first thing from Kerbin to ever touch this world. The data coming in is... I need a minute. This is a lot. }
	}
```

- [ ] **Step 3: Add to BKEX_CrewedOrbit (accept)**

Insert before the final `}` of BKEX_CrewedOrbit:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Crewed Orbit

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = The probe did its job — now we send the crew. Bring them there, bring them home. That is the mission. That is always the mission. }
	}
```

- [ ] **Step 4: Add to BKEX_CrewedLanding (accept)**

Insert before the final `}` of BKEX_CrewedLanding:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Boots on the Ground

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle_sadA
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = This is the one that makes the history books. Kerbals on a new world. Bring them home safe — everything else is secondary. }
	}
```

- [ ] **Step 5: Add to BKMC_FirstRelay (accept)**

Insert before the final `}` of BKMC_FirstRelay:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Signal Strength

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = A relay satellite is a mirror for radio signals. Without them, every probe that flies behind a planet goes silent. Communication is the backbone of exploration — we build the network first, then we explore. }
	}
```

- [ ] **Step 6: Read and add to RelayConstellation (accept)**

Read `RelayConstellation.cfg` and add:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Network Coverage

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = One relay has blind spots. Three relays, evenly spaced, and nothing hides from us. Full coverage — every signal finds a path home. }
	}
```

- [ ] **Step 7: Verify syntax for all modified files**

---

## Task 15: Tier 2 — Aviation Milestones

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/Mach3.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/Hypersonic.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/HighSpeedLowPass.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/EdgeOfSpace.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/IslandHop.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/DessertRun.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/WoomerangDash.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/OrbitalSpaceplane.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/SpaceplaneDocking.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/LaytheSSTO.cfg`

Quick one-liners. Read each file, add one BEHAVIOUR before the final `}`.

- [ ] **Step 1: Read all 10 files to find insertion points**

- [ ] **Step 2: Add to Mach3 (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Mach 3

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = true_nodA
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = Mach 3 — three times the speed of sound. At this velocity, the air itself becomes the enemy. Friction heats the airframe until the leading edges glow. The SR-71 Blackbird cruised at this speed. Our pilot just joined very exclusive company. }
	}
```

- [ ] **Step 3: Add to Hypersonic (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Hypersonic

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_thumbsUp
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = Mach 5. Hypersonic. At this speed, the air in front of the aircraft cannot get out of the way fast enough — it compresses into a shockwave that heats everything to plasma temperatures. And our pilot flew straight through it. }
	}
```

- [ ] **Step 4: Add to HighSpeedLowPass (accept)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Buzzing the Tower

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = The ground crew wants a flyby. Fast and low — below a thousand meters. I am told this is for "calibration purposes." I suspect it is for bragging rights. Please do not hit anything. }
	}
```

- [ ] **Step 5: Add to EdgeOfSpace (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Edge of Space

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = Eighteen kilometers up — where the atmosphere thins to almost nothing and the sky darkens to near-black. This is the boundary between aviation and spaceflight. You are flying at the edge of where wings stop working. }
	}
```

- [ ] **Step 6: Add to IslandHop (accept)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Cross-Country

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = There is an airfield on the island east of KSC. Short hop over the water. Navigation practice — follow the heading and keep the altitude steady. Try to land on the actual runway. }
	}
```

- [ ] **Step 7: Add to DessertRun (accept)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Dessert Airfield

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = There is an old airstrip out in the desert. Dusty, flat, and a long way from help. Good place to test if our cross-country navigation is as good as we think. }
	}
```

- [ ] **Step 8: Add to WoomerangDash (accept)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Woomerang

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gus
		characterAModel = Strategy_MechanicGuy
		characterAAnimation = idle
		characterAColor = #FFFFC078

		LINE { speaker = A; text = Woomerang is up north — way up. Cold, windy, and the runway is more of a suggestion. She will handle it, but keep the approach speed low. I just waxed her. }
	}
```

- [ ] **Step 9: Add to OrbitalSpaceplane (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Orbital SSTO

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gus
		characterAModel = Strategy_MechanicGuy
		characterAAnimation = true_smileA
		characterAColor = #FFFFC078

		LINE { speaker = A; text = She made it to orbit and came back in one piece. I was not sure about the thermal tiles on the belly, but she held together. Runway to orbit and back — that is engineering. }
	}
```

- [ ] **Step 10: Add to SpaceplaneDocking (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Docking

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_nodA
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = A spaceplane that can dock with a station. That is not just a spacecraft — that is a reusable shuttle. The logistics of space just got a whole lot simpler. }
	}
```

- [ ] **Step 11: Add to ExoFlight (accept)**

Read `LaytheSSTO.cfg` first, then add:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Alien Skies

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = An atmosphere. Not ours — an alien one. Different composition, different pressure, different temperature. But if the air is thick enough... wings will work. Flying on another world. Think about that. }
	}
```

- [ ] **Step 12: Verify syntax for all 10 files**

---

## Task 16: Tier 2 — Kerbin Side Bases (19 contracts)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/KS_Batch1.cfg` (4 contracts)
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/KS_Batch2.cfg` (4 contracts)
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/KS_Batch3.cfg` (4 contracts)
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/KS_Batch4.cfg` (7 contracts)

Each base gets one Gene accept line with light flavor about the destination.

- [ ] **Step 1: Read all 4 batch files**

Read each file to identify CONTRACT_TYPE names, closing braces, and base names.

- [ ] **Step 2: Add ChatterBox to each contract in KS_Batch1**

For each of the 4 contracts, insert before the final `}`:

Template (adjust `title` and `text` per base):

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = [Base Name]

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		LINE { speaker = A; text = [One-liner about this specific base location and what makes it interesting] }
	}
```

Write unique one-liners per base. Examples:
- Kola Island: "Kola Island — short hop south over the water. Good warmup for longer trips."
- Jeb's Junkyard: "Jeb has a... personal collection of aerospace artifacts out east. He calls it a junkyard. I call it a liability. Either way, there is a flat spot to land."
- Meeda: "Meeda Station, inland. Longer flight, real navigation required. Keep an eye on your fuel."
- Kojave: "Kojave is out in the desert flats. Good visibility, long approach, and not much to hit. Which is reassuring."

- [ ] **Step 3: Add ChatterBox to each contract in KS_Batch2**

Same pattern, 4 contracts. Unique one-liners:
- Baikerbanur: "Baikerbanur — the other space center. They have been launching longer than us, though they will not admit their success rate."
- Cape Kerman: "Cape Kerman, on the eastern coast. Beautiful approach over the water if you keep your altitude up."
- Uberdam: "Uberdam is north, past the mountains. The approach is tricky with crosswinds, so take it slow."
- Sandy Island: "Sandy Island. It is sandy. It is an island. The runway is optimistically short."

- [ ] **Step 4: Add ChatterBox to each contract in KS_Batch3**

Same pattern, 4 contracts:
- Kerman Atoll: "Kerman Atoll — a speck in the ocean with a landing strip that was clearly designed by someone who has never landed an airplane."
- South Field: "South Field, deep in the southern hemisphere. Long flight, make sure you packed enough fuel. And Snacks."
- Harvester: "Harvester Station, near the mining operations. Industrial area — the approach has some tall structures, so stay alert."
- Kamberwick Green: "Kamberwick Green. Quiet, pastoral, and suspiciously far from anything. I think the staff transferred here voluntarily."

- [ ] **Step 5: Add ChatterBox to each contract in KS_Batch4**

Same pattern, 7 contracts:
- Polar Research Alpha: "Polar Research Alpha. It is cold, it is dark half the year, and the runway is made of packed ice. The crew there is... eager for visitors."
- South Lake: "South Lake — near the southern crater lake. Beautiful scenery, challenging approach through the valley."
- Round Range: "Round Range, military testing ground. Land where they tell you to land and do not ask about the scorch marks."
- Nye Island: "Nye Island, way out in the middle of the ocean. Navigation is everything — there are no landmarks until you see the runway."
- Dununda: "Dununda, in the outback. Flat, dry, and an absurdly long runway. If you overshoot here you were not trying."
- Hazard Shallows: "Hazard Shallows. The name says it all — shallow waters, strong crosswinds, and a runway that floods at high tide. Allegedly."
- Kermundsen: "Kermundsen Station, near the south pole. The farthest base on our network. Getting there is the challenge — getting back is the real test."

- [ ] **Step 6: Verify syntax for all 4 batch files**

---

## Task 17: Tier 2 — Anomaly Surveyor (Replace DialogBox)

**Files:**
- Modify: 15 AnomalySurveyor contract files (listed below)

Replace all existing `BEHAVIOUR { type = DialogBox ... }` blocks with ChatterBox equivalents. Keep all other BEHAVIOURs (WaypointGenerator, etc.) untouched.

Each gets a Tier 2 complete reaction from Linus or Gene.

- [ ] **Step 1: Read all anomaly files that have DialogBox**

Read these files to understand their existing DialogBox structures:
- `Kerbin_Pyramids.cfg`, `Kerbin_UFO.cfg`, `Kerbin_IslandAirfield.cfg`
- `Kerbin_Monolith_Additional.cfg`
- `Mun_Monolith.cfg`, `Mun_Memorial.cfg`, `Mun_RockArch.cfg`, `Mun_UFO.cfg`
- `Bop_DeadKraken.cfg`, `Duna_Face.cfg`, `Duna_MSL.cfg`
- `Moho_Mohole.cfg`, `Tylo_Cave.cfg`, `Vall_Icehenge.cfg`, `Jool_Monolith.cfg`

- [ ] **Step 2: Replace DialogBox blocks in each file**

For each file:
1. Delete the entire `BEHAVIOUR { type = DialogBox ... }` block (including all DIALOG_BOX sub-nodes)
2. Insert a ChatterBox BEHAVIOUR with `onState = CONTRACT_SUCCESS` (or `PARAMETER_COMPLETED` matching the original trigger)

Template:
```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = [Discovery Title]

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = [1-2 lines reacting to the specific discovery — awe, mystery, humor appropriate to the anomaly] }
	}
```

Write unique reaction lines for each anomaly. The existing DialogBox text can inform the tone but should be rewritten in Linus's voice. Examples:
- Kerbin Pyramids: "An ancient kerbal civilization... buried in the sand. The inscription reads 'Look on my works, ye Mighty, and despair.' Someone had a flair for the dramatic."
- Mun UFO: "That is... definitely not one of ours. The design is completely foreign. Wernher is going to lose his mind when he sees this data."
- Bop Dead Kraken: "It is enormous. And it is dead. I think. I hope. The readings are unlike anything biological we have ever recorded. What was it doing out here?"
- Duna Face: "It looks like a face. Wernher says it is just erosion patterns. I am not entirely convinced."

**Note:** Some files have multiple DIALOG_BOX nodes that fire on PARAMETER_COMPLETED. These were sequential pop-ups in stock CC. Replace the entire DialogBox BEHAVIOUR (containing all DIALOG_BOX nodes) with a single ChatterBox BEHAVIOUR. ChatterBox handles multi-line dialogue natively through LINE nodes.

- [ ] **Step 3: Handle Kerbin_Monolith_Additional.cfg carefully**

This file has additional monolith contracts (multiple CONTRACT_TYPEs). Each has its own DialogBox. Replace each independently with a ChatterBox BEHAVIOUR inside its own CONTRACT_TYPE.

- [ ] **Step 4: Handle Jool_Monolith.cfg**

This file has two DIALOG_BOX nodes — one on PARAMETER_COMPLETED for the initial discovery, one on a second parameter. Combine into a single ChatterBox BEHAVIOUR with multiple LINEs.

- [ ] **Step 5: Verify syntax for all 15 files**

---

## Task 18: Tier 2 — OPM and Kcalbeloh Remaining (5 contracts)

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/OPM_Tekto_Spire.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/OPM_Thatmo_Monolith.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/OPM_Plock_Monolith.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/Kcalbeloh_Beyond.cfg`
- Modify: `GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/Kcalbeloh_Dipuc.cfg`

These are deep-space anomalies. Linus for OPM, Wernher for Kcalbeloh.

- [ ] **Step 1: Read all 5 files**

- [ ] **Step 2: Add to OPM_Tekto_Spire (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The Spire

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = An orange crystalline spire, rising out of Tekto's haze. The geomagnetic readings are extraordinary — this structure is interacting with Tekto's magnetic field in ways I cannot explain. It should not exist. But here it is. }
	}
```

- [ ] **Step 3: Add to OPM_Thatmo_Monolith (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Frozen Signal

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = Another monolith, frozen into the surface of Thatmo. The signal is stronger than any we have detected before. It is pointing outward — past Neidon, past Plock, toward the edge of everything. Something wants us to keep going. }
	}
```

- [ ] **Step 4: Add to OPM_Plock_Monolith (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Edge of the System

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		LINE { speaker = A; text = The last monolith. The edge of our solar system. And the signal... it is not pointing at another planet this time. It is pointing at a specific region of empty space. Except I do not think it is empty. Wernher needs to see these readings immediately. }
	}
```

- [ ] **Step 5: Add to Kcalbeloh_Beyond (accept)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Through the Door

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = Our probe confirmed the wormhole is traversable. On the other side — a new star system, orbiting something we cannot see directly. The instruments say it is there. A presence defined by what it does to everything around it. We go through. }
	}
```

- [ ] **Step 6: Add to Kcalbeloh_Dipuc (complete)**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Gravitational Anomaly

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		LINE { speaker = A; text = The gravitational readings on Dipuc are... wrong. Beautifully wrong. Something beneath the surface is bending spacetime in ways our models cannot account for. Another piece of the puzzle. Whoever built the monoliths left something here too. }
	}
```

- [ ] **Step 7: Verify syntax for all 5 files**

---

## Task 19: Tier 2 — RoverSurvey

**Files:**
- Modify: `GameData/BadgKatCareer/ContractPacks/Milestones/RoverSurvey.cfg`

- [ ] **Step 1: Read RoverSurvey.cfg and add accept scene**

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Surface Survey

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Gus
		characterAModel = Strategy_MechanicGuy
		characterAAnimation = idle
		characterAColor = #FFFFC078

		LINE { speaker = A; text = Science team has marked some waypoints they want samples from. Take the rover out, hit the spots, run the experiments. She knows the way — just keep her rubber side down. }
	}
```

- [ ] **Step 2: Verify syntax**

---

## Task 20: Final Verification

- [ ] **Step 1: Count all ChatterBox BEHAVIOUR nodes added**

Grep across all BadgKatCareer contract files for `type = ChatterBox` and count. Expected: ~89 total.

```bash
grep -r "type = ChatterBox" "GameData/BadgKatCareer/ContractPacks/" | wc -l
```

- [ ] **Step 2: Verify no remaining DialogBox nodes in modified files**

```bash
grep -r "type = DialogBox" "GameData/BadgKatCareer/ContractPacks/AnomalySurveyor/" | wc -l
```

Expected: 0 (all replaced with ChatterBox).

- [ ] **Step 3: Syntax check — balanced braces**

For each modified file, verify opening and closing braces match. Quick check:

```bash
for f in GameData/BadgKatCareer/ContractPacks/**/*.cfg; do
  opens=$(grep -c '{' "$f")
  closes=$(grep -c '}' "$f")
  if [ "$opens" != "$closes" ]; then
    echo "MISMATCH: $f (open=$opens close=$closes)"
  fi
done
```

- [ ] **Step 4: Launch KSP and verify**

1. Check `KSP.log` for ChatterBox or ContractConfigurator errors
2. Start new career, accept FirstScience — verify ChatterBox popup appears with Gene + Wernher
3. Complete FirstScience — verify completion popup
4. Accept FirstRover — verify Gus appears
5. Check Mission Control for contract availability (no broken requirements)

---

## Summary

| Task | Files | Tier | Contracts | Scenes Added |
|---|---|---|---|---|
| 1 | FirstSteps.cfg | 1 | FirstScience | 2 (accept + complete) |
| 2 | FirstRover.cfg | 1 | FirstRover | 2 |
| 3 | FirstFlight.cfg | 1 | FirstFlight | 2 |
| 4 | Kerbin_Monolith.cfg | 1 | Kerbin_Monolith | 2 (replace DialogBox) |
| 5 | FirstSteps.cfg | 1 | UnmannedSuborbital | 2 |
| 6 | EarlyCrewed.cfg | 1 | CrewedUpperAtmo | 1 |
| 7 | UnmannedOrbit.cfg | 1 | UnmannedOrbit | 2 |
| 8 | EarlyCrewed.cfg | 1 | CrewedSuborbital | 2 |
| 9 | CrewedExploration.cfg | 1 | CrewedHomeOrbit | 2 |
| 10 | ProbeExploration.cfg | 1 | ProbeFlyby | 2 |
| 11 | SoundBarrier.cfg | 1 | SoundBarrier | 1 |
| 12 | SuborbitalSpaceplane.cfg | 1 | SuborbitalSpaceplane | 1 |
| 13 | Kcalbeloh_Wormhole/BlackHole | 1 | Wormhole + BlackHole | 2 |
| 14 | Multiple | 2 | 6 exploration/relay | 6 |
| 15 | Multiple | 2 | 10 aviation/SSTO | 10 |
| 16 | KS_Batch1-4.cfg | 2 | 19 bases | 19 |
| 17 | 15 anomaly files | 2 | ~15 stock anomalies | ~15 (replace DialogBox) |
| 18 | 5 OPM/Kcalbeloh files | 2 | 5 outer system | 5 |
| 19 | RoverSurvey.cfg | 2 | RoverSurvey | 1 |
| 20 | — | — | Verification | 0 |
| **Total** | | | | **~77 scenes** |
