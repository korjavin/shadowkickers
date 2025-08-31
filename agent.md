Coding Agent Task: 2D Platformer Game - "Crime Kickers vs The Bad Guys"
Objective
Create a single-file, browser-based 2D platformer game using HTML, CSS, and vanilla JavaScript. The game's core mechanic revolves around switching between four distinct characters to solve environmental puzzles.

Core Concept
The game is inspired by real-time tactics games like Shadow Tactics or Commandos, but adapted into a 2D side-scrolling platformer. The player must strategically use the unique abilities of four different heroes to navigate a level they could not pass alone.

Technical Requirements
Single-File Architecture: The entire game (HTML, CSS, JavaScript) must be contained within a single .html file.

Rendering: Use the HTML <canvas> element for all game rendering.

Styling: Use Tailwind CSS for the UI elements (character selector, messages).

Camera: Implement a basic side-scrolling camera that follows the currently active player.

UI:

Create a simple UI element that displays the four available characters, highlighting the one currently selected.

Display on-screen tutorial text prompts that appear when the player reaches certain parts of the level.

Create a message box for win/lose conditions.

Character Specifications & Controls
Global Controls:

Move: Arrow Keys (Left, Right)

Jump: Up Arrow

Switch Character: 1 (Mister Underpants), 2 (Windman), 3 (Teibi), 4 (Primm). When switching, the new character should appear at the old character's location.

Primary Ability: Spacebar

Character 1: Mister Underpants

Appearance: A simple rectangle of a distinct color (e.g., red) with a small cape.

Ability 1 (Fly): When in the air, holding the Up Arrow key should slow his descent and allow for a short glide. This should consume a regenerating "flight fuel" resource.

Ability 2 (Shoot): Pressing Spacebar fires a projectile forward that can eliminate an enemy.

Character 2: Windman

Appearance: A simple rectangle of a distinct color (e.g., light blue).

Ability (Wind Gust): Pressing Spacebar near a specific "wind-receptive" object should activate it. For Level 1, this will be a lift platform that moves upwards.

Character 3: Teibi

Appearance: A simple rectangle of a distinct color (e.g., green).

Ability (Shrink): Pressing Spacebar toggles his size between normal and small. In his small form, he can fit through narrow horizontal or vertical gaps.

Character 4: Primm

Appearance: A simple rectangle of a distinct color (e.g., purple).

Ability 1 (Dash): Pressing Spacebar executes a fast, short-range forward dash that can defeat an enemy.

Ability 2 (Phase): Holding the C key allows him to pass through designated "phase walls" unharmed. Other characters should treat these walls as solid.

First Level Design
Create a linear, side-scrolling level that sequentially requires the use of each character's abilities to progress from left to right.

Start Zone: A basic platform to start on.

Teibi Puzzle: An impassable wall with a small gap at the bottom. The player must switch to Teibi, shrink, and walk through the gap.

Mister Underpants Puzzle: A large horizontal chasm that is too wide to jump across. The player must switch to Mister Underpants and use his flight/glide ability to cross. Place one basic patrolling enemy on the other side that the player can shoot.

Primm Puzzle: The path is blocked by a vertical "phase wall" (give it a distinct, semi-transparent look). The player must switch to Primm and hold C to walk through it. Place another patrolling enemy after the wall that can be defeated with his dash.

Windman Puzzle: The goal is on a high ledge that is unreachable by jumping. Below the ledge, place a "wind platform". The player must switch to Windman, stand near the platform, and use his ability to raise it.

Goal: A distinct object or zone at the end of the Windman puzzle. Reaching it should display a "Level Complete!" message.