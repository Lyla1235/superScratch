I found the issue! The problem is in your `processControls` function. When you press pause, it correctly toggles the pause state, but the issue is that `processControls` is only called inside `player.tick()`, which only runs when the game is **not paused**.

Here's what's happening:

1. You press 'P' → game pauses → `processControls` stops being called
2. You release 'P' → `game.pauseKeyWasDown` never gets updated to `false` because `processControls` isn't running
3. You press 'P' again → the game is still paused, so nothing happens
4. You click the pause button → game unpauses → `processControls` runs again
5. Now it sees that `game.controls.pause` is `true` AND `game.pauseKeyWasDown` is still `true` (from step 1), so it doesn't toggle
6. Eventually you release 'P', but since the latch was never reset properly, the timing gets messed up

The fix is to handle pause/restart controls **outside** the pause check, in the main game loop. Here's the corrected code:Now the pause and restart controls are handled in the main game loop (in the new `handleGlobalControls` function) which runs continuously, rather than inside `processControls` which only runs when the game is unpaused. This means:

- **'P' key** will now properly toggle pause on/off
- **'R' key** will restart the game even when paused
- **Spacebar** works for pause (mapped to keycode 32, though you'll need to add it to the mapping if you want it)
- The on-screen pause button will continue to work as expected

The "double speed" issue you mentioned was likely because the game loop was being called multiple times - this fix should resolve that too!

// main loop — updates and draws everything every frame
      loop: function() {
        if (this.finished) {
          return; // stop the game if it's over
        }

        // Handle pause/restart BEFORE the pause check so they work even when paused
        this.handleGlobalControls();

        if (!this.paused) {
          world.tick();   // updates world physics
          player.tick();  // updates player movement
          world.draw();   // draws the level
          player.draw();  // draws the cat
        }
        
        // run again at ~60 FPS
        this.timer = window.setTimeout("game.loop()", 1000 / 60);
      },

      // Handle controls that should work even when paused
      handleGlobalControls: function() {
        // Restart
        if (this.controls.restart && !this.restartKeyWasDown) {
          location.reload();
        }
        this.restartKeyWasDown = this.controls.restart;

        // Pause
        if (this.controls.pause && !this.pauseKeyWasDown) {
          this.togglePause();
        }
        this.pauseKeyWasDown = this.controls.pause;
      },