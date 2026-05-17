# Frame Analysis Brief Template

Use this when delegating frame analysis to a subagent. Populate the `<placeholders>` before spawning.

```
Analyze the gameplay frames at <frames_dir> to build an accurate stat timeline for the full <duration>s video.

Game: <game_name> (e.g., "COD Mobile Team Deathmatch on Shipment", "COD Mobile Zombies on Shi No Numa")
Mode: <mode> (e.g., "Team Deathmatch", "Domination", "Zombies Classic")

The frames are named frame_XXXs.jpg where XXX is the timestamp in seconds.
Frames at: <list every N seconds, e.g. 0, 3, 6, 9, ..., DURATION>

For each frame, look at the HUD for:
1. Core stat to track: <kills / round / score / players left>
2. Secondary stats: <deaths, points, zombies remaining, weapon>
3. Medals/notifications: <e.g., "First Blood, Double Kill, Bloodthirsty" or "Round Complete, Death">
4. Team/match score from top-center display

Build a JSON timeline like:
[
  {"time": 0, "<core_stat>": 0, "<secondary_stat>": 0, "event": "Match starting"},
  {"time": 27, "<core_stat>": 1, "<secondary_stat>": 0, "event": "MEDAL NAME"},
  ...
]

Also identify:
- Total <core_stat> at match end
- All death timestamps (when killcam/revive screen appears)
- Final score shown
- Mode-specific data: round numbers, weapon changes, essence/points for Zombies

Write the timeline to <output_path>. Return a summary of the match arc and key events.
```
