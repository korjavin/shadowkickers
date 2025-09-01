# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"Crime Kickers vs The Bad Guys" is a single-file, browser-based 2D stealth-puzzle platformer game. The core mechanic involves switching between four unique characters with distinct abilities to overcome obstacles and solve environmental puzzles.

## Architecture

### Single File Structure
- **Main File**: `the_unlikely_squad.html` - Contains the complete game (HTML, CSS, JavaScript)
- **Audio Assets**: `Verse 1.mp3` and `Verse 1 (2).mp3` - Background music tracks

### Game Engine Components
- **Game Class**: Main game engine handling physics, rendering, collision detection
- **Character System**: Four playable characters with unique abilities
- **Level System**: Two levels with increasing difficulty
- **Difficulty Modes**: Easy (3 hearts), Normal, Hardcore (more enemies)

## Development Commands

This is a static HTML game with no build system. To develop:

1. **Run the game**: Open `the_unlikely_squad.html` in any modern web browser
2. **Test changes**: Refresh the browser after editing the HTML file
3. **Debug**: Use browser developer tools console for JavaScript debugging

## Character System

### Core Characters
1. **Mister Underpants (Red)**: Flight and projectile shooting
2. **Windman (Blue)**: Wind manipulation for platforms and enemy control
3. **Teibi (Green)**: Size transformation to access tight spaces
4. **Primm (Purple)**: Phasing through walls and dash attacks

### Character Switching
- Characters share position when switching (teleport mechanic)
- Only one character active at a time
- Each character has unique collision rules and abilities

## Game Physics

### Movement System
- Gravity: 0.8 units/frame
- Friction: 0.8 multiplier
- Jump power: 15 units
- Character speed: 5 units/frame

### Collision Detection
- Platform collision with ground detection
- Enemy collision with stomp vs. damage mechanics
- Special collision rules for shrunk Teibi and phasing Primm

## Level Design

### Level 1 Structure
- Single level with linear progression requiring each character's abilities
- Sections: Starting area → Teibi puzzle → Flight section → Primm phase walls → Wind platform finale
- Game complete after reaching the goal

## Enemy System

### Enemy Behavior
- Patrol movement between defined boundaries
- Player interaction: stomp to kill, touch sides/below for damage
- Wind effects: can be blown away and stunned temporarily
- Dash attacks: instantly eliminated by Primm's dash

### Difficulty Scaling
- **Easy**: Hearts system with respawn
- **Normal**: Standard enemy count, no extra lives
- **Hardcore**: 3x enemy density across all sections

## UI Components

### Character Selection
- Top navigation bar with character portraits
- Visual feedback for active character
- Click or keyboard shortcuts (1-4 keys)

### Tutorial System
- Context-sensitive messages when switching characters
- Automatic display with 3-second timeout
- Character-specific ability explanations

### Audio System
- Background music rotation between two tracks
- Volume controls and mute functionality
- Automatic track switching every 60 seconds

## Code Organization

### Main Classes
- `Game`: Primary game loop, physics, rendering
- Character objects: Stored in `characters` array with ability properties
- Platform/Object arrays: Level geometry and interactive elements

### Key Methods
- `initLevel()`: Level-specific platform and enemy setup
- `switchCharacter()`: Character swapping with position inheritance
- `useAbility()`: Character-specific ability activation
- `checkCollisions()`: Physics and interaction detection

## Testing the Game

### Basic Testing
1. Test all four character abilities work correctly
2. Verify level progression requires each character's unique ability
3. Check enemy interactions (stomp vs. damage)
4. Validate win/lose conditions

### Browser Compatibility
- Tested on Chrome, Firefox, Edge
- Requires modern browser with Canvas API support
- Audio may require user interaction to start (browser autoplay policies)

## Known Game Features

### Special Mechanics
- **Flight Fuel**: Mister Underpants has limited flight time that regenerates on ground
- **Wind Particles**: Visual feedback for Windman's ability activation
- **Phasing Effect**: Visual transparency when Primm phases through walls
- **Collectible Coins**: Optional collectibles scattered throughout levels

### Accessibility
- Keyboard-only controls (no mouse required for gameplay)
- Visual indicators for character abilities and states
- Clear tutorial messages explaining character functions