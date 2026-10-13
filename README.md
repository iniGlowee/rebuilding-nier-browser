# Rebuilding NieR:Automata's systems in a browser engine

*A devlog about recreating the feel of NieR:Automata's gameplay systems inside a
browser three.js engine — motion matching, pod combat, the flight unit, dialogue,
cover — written from the research behind the AE1 ("Angel's Egg") engine. The systems
and the journey, not the content.*

This is a **process** write-up. It contains **no extracted game data** — no models,
animations, textures, audio, scripts or names. Every tool it references reads **your
own legally-owned files**. The interesting part was never the assets anyway; it was
rebuilding the *systems* that make the game move the way it does.

---

## The thesis: port the systems, not the content

NieR:Automata is a character-action game stitched onto a world that keeps changing
genre — third-person melee, twin-stick shooter, side-scroller, bullet-hell, text
adventure. What makes it *feel* like itself isn't any one asset; it's a set of systems:
locomotion that blends fluidly, a pod that fires independently of where you're facing, a
transforming flight unit, and dialogue/cover layers that frame the action. Rebuild those
systems faithfully in a browser engine and you can drive them with placeholder or
clean-room content and it still reads as "that game."

So the project split cleanly: a **content pipeline** (get a character or animation out of
the game's files, into glTF, into the engine — strictly for personally-owned assets,
code released, bytes never) and a **systems layer** (the reusable gameplay runtime,
which is the genuinely transferable part).

## 1. Locomotion → motion matching in the browser

The standout movement system is **motion matching**: instead of a hand-wired blend tree
of "idle → walk → run," you keep a **library of short animation clips** and, every frame,
pick the pose+clip whose *future trajectory* and *current pose* best match where the
player is actually trying to go. The result is responsive locomotion with no visible state
machine.

Rebuilding it in the browser meant:

- **Feature vectors** per animation frame — foot positions/velocities, hip velocity, and a
  short **future trajectory** (where this clip is about to go over the next N frames).
- **A query** each tick built from the desired trajectory (from input) and the current
  pose.
- **A nearest-match search** over the library's feature vectors, with a bias to stay on
  the current clip (so it doesn't jitter between near-ties), then a short **cross-fade**
  into the chosen clip.
- **A curator UI** to load a clip library, visualise the features, and tune the weights —
  because motion matching lives or dies on how you weight pose-vs-trajectory.

> This runtime is a candidate standalone release (`browser-motion-matching`): the engine
> and a **synthetic / CC0 clip set** can ship; a game's clips never do. The runtime is
> content-agnostic by design — that's the whole point.

**Trap — clips arrive retargeted.** The library came from a different skeleton than the
in-engine character. Motion matching assumes one consistent rig; every clip has to be
**retargeted onto the engine skeleton first**, or the feature vectors aren't comparable
and the search picks nonsense. Retarget is a pipeline step, not a runtime hack.

## 2. Content pipeline → animations and models, headless

Feeding the systems meant decoding the game's animation (`.mot`) and model formats
headlessly — no original toolchain — and baking them to animated glTF the engine loads.
Two tools came out of this and were released early (pointed at the user's own files):

- a **bulk `.mot` animation extractor**, and
- an **enemy model + animation-pack → engine-ready animated glTF baker** that does a
  **no-retarget bake** (keep the source rig, bake the clips onto it) so enemy animations
  come through without a retarget round-trip.

The hard-won lesson from the facial/animation decode work (shared in more detail in a
separate method devlog): **when the output is wrong, the decode is wrong.** Melting faces
and drifting limbs were **keyframe-codec** bugs — wrong axis order / bit-width on a
compressed rotation track — not skinning or retargeting, which is where everyone looks
first. You find the maintained community reference for the format, transcribe its codec
tables **verbatim**, and judge the result against a ground-truth render. No downstream
band-aids.

## 3. Pod combat → a weapon that aims itself

The **pod** is a floating drone that fires continuously in a direction *decoupled* from
the character's facing — you move and dodge with one set of inputs while the pod streams
fire at a locked or aim-assisted target. Rebuilding it is a lesson in **separating the
weapon from the wielder**:

- the pod is its own entity with its own transform, orbiting/trailing the character;
- its aim is a separate vector (nearest target, or free-aim), not the character's forward;
- firing is a cooldown-gated spawn of projectiles with their own lifetime/collision;
- weapon *families* (light/heavy/ranged) are a **data table**, not branching code — swap
  the row, get a different cadence/recoil/projectile.

This "weapon as a table-driven entity" pattern turned out to generalise far beyond this
game and became a reusable engine element.

## 4. The flight unit → one entity, several genres

The transforming **flight unit (Ho229-style)** is where the game flips between
third-person flight, twin-stick, and side-scroller. The trick is **one vehicle entity with
swappable control + camera modes**, not three separate minigames:

- the flight model (thrust, banking, strafe) stays constant;
- a **mode** selects the camera rig (chase / top-down / locked-plane) and the input
  mapping;
- transitions are camera/mode changes over the *same* entity, so momentum carries through
  the genre flip instead of hard-cutting.

Building it this way meant the "shoot-em-up section" wasn't a bespoke level — it was the
flight entity in top-down mode on a scripted path.

## 5. Dialogue + cover → the framing layers

Two smaller systems round out the feel:

- **Dialogue.** A UI-lock that idles the character, a **DOM-overlay** dialogue box (text,
  speaker, a custom cursor and period-appropriate font) drawn *above* the 3D canvas rather
  than into it, and per-NPC trees fired from an interact hook. Keeping dialogue in the DOM
  means real text layout and fonts for free, and the GL context never fights a menu.
- **Cover.** Snap-to-cover against world geometry: detect a cover edge near the character,
  bind the character to it, and switch the camera/aim to peek-and-fire — another
  **state-on-an-entity** pattern rather than a separate mode of the whole game.

## 6. The through-line: systems are entities with tables and modes

Every system above resolves to the same two ideas:

1. **Model behaviour as data tables** (weapon families, enemy actions, locomotion
   features) so new behaviour is a new row, not new branching.
2. **Model complex entities as one object with swappable modes** (the flight unit, cover,
   dialogue-lock) so transitions preserve state instead of hard-cutting between bespoke
   subsystems.

That's what made the port *tractable* in a browser engine, and it's why the systems layer
is the reusable part — it carries to any character-action project, with none of the
original content attached.

## IP discipline — why only the systems ship

NieR:Automata's models, animations, audio and text are Square Enix / PlatinumGames
copyrighted content. The discipline that keeps this shareable:

- **Ship code and knowledge, distribute no assets.** The *systems* (motion-matching
  runtime, weapon/enemy table patterns, flight/camera modes) and the *decoder tools* are
  releasable; the game's models, clips, audio and names stay out of git and out of releases.
- **The user brings their own files.** Decoders and bakers point at assets the user legally
  owns.
- **Clean-room content for demos.** The motion-matching runtime ships with synthetic / CC0
  clips, never the game's.
- **No affiliation implied.** This is independent research, not affiliated with or endorsed
  by the rights-holders.

---

## In one breath

Decode the character/animation formats headlessly (fix the *decode*, never band-aid
downstream) → bake to glTF for your own files only → drive a **content-agnostic systems
layer**: motion-matched locomotion, a pod that aims itself, a flight unit that is one
entity in several camera modes, DOM-overlay dialogue, snap-to-cover — everything a data
table or a mode on an entity, so the systems carry to the next project while the content
stays with its owner.

---

*From the AE1 ("Angel's Egg") browser three.js engine project's NieR:Automata research.
Systems and journey only — no models, animations, audio, text, or other game assets
included. Released under [MIT](LICENSE) (covers the prose and any snippets).
NieR:Automata is a trademark of its respective owners; this project is independent,
unaffiliated and not endorsed.*
