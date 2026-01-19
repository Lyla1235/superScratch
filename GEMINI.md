You have run into three specific issues that are interacting with each other to cause this behavior:

1. **The Catch-22 Loop:** You check for key presses inside your game loop. When you pause the game, you stop the loop. Because the loop is stopped, the game stops checking the keyboard. Therefore, the game cannot "hear" you press the key to resume.
2. **The Ghost Loop (Double Speed):** When you pressed 'P', your code didn't actually stop the `setTimeout` loop correctly; it kept scheduling new frames in the background that did nothing (because `paused` was true). When you clicked the on-screen button, you started a *new* loop. The old "ghost" loop saw `paused` was false and started working again. Two loops running at once = double speed.
3. **Missing Map:** The Space Bar (Key code 32) simply wasn't in your key mapping list.

Here is the fixed code. I have updated the `game` object logic to fix the loop and moved the pause detection so it works even when the game is stopped.

### The Fix

Replace the `<script>` section in your file with this updated version:

```html
  <script type="text/javascript">
    
    // this is the setup for the beginning of the game
    // it creates the context and sets up controls.
    var canvas = document.getElementById("game");
    var ctx = canvas.getContext("2d");

    var game = {
      paused: false,
      timer: null,
      finished: false,
      restartKeyWasDown: false,

      // keyboard input manager
      controls: {
        left: false,
        right: false,
        up: false,
        down: false,

        // maps keycodes to actions - ASCII values for A, D, W, S
        mapping: {
          65: "left",
          68: "right",
          87: "up",
          83: "down",
          80: "pause",    // 'P' key
          32: "pause",    // ADDED: Space Bar
          82: "restart"
        },

        buttonPress: function(keyInfo) { 
          const action = this.mapping[keyInfo.keyCode];
          
          // FIX: Handle pause immediately here, outside the game loop
          if (action === "pause") {
            game.togglePause();
            // Update the HTML button text to match
            document.getElementById("pauseBtn").textContent = game.paused ? "Resume" : "Pause";
            return; 
          }

          if (action) this[action] = true; 
        }, 
        buttonRelease: function(keyInfo) { 
          const action = this.mapping[keyInfo.keyCode];
          if (action && action !== "pause") this[action] = false; 
        },

        // hooks the controls into the browser
        connect: function() {
          window.addEventListener("keydown", function(keyInfo) {
            game.controls.buttonPress(keyInfo);
          }, false);

          window.addEventListener("keyup", function(keyInfo) {
            game.controls.buttonRelease(keyInfo);
          }, false);
          
          // Prevent scrolling with arrow keys and space
          window.addEventListener("keydown", e => {
            if (["Space", "ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(e.code)) {
              e.preventDefault();
            }
            // Also prevent Space from clicking the focused button (which causes double toggles)
            if (e.keyCode === 32) {
               e.preventDefault();
            }
          });
        }
      },

      // plays game audio (jump sound + background music)
      sounds: {
        enabled: true,
        jump: function() { this.play("meow.wav"); },
        backgroundMusic: function() { this.play("background.mp3"); },
        play: function(filename) {
          if (this.enabled) {
            // Note: Audio might be blocked by browser until user interacts
            try { new Audio("Sounds/" + filename).play(); } catch(e){}
          }
        }
      }, 

      
      // main loop — updates and draws everything every frame
      loop: function() {
        // FIX: If paused, STOP here. Do not schedule the next frame.
        // This prevents the "Ghost Loop" that causes double speed later.
        if (this.finished || this.paused) {
          return; 
        }

        world.tick();   // updates world physics
        player.tick();  // updates player movement
        world.draw();   // draws the level
        player.draw();  // draws the cat
        
        // run again at ~60 FPS
        this.timer = window.setTimeout("game.loop()", 1000 / 60);
      },

      // toggles the pause state of the game
      togglePause: function () {
        this.paused = !this.paused;

        // If we just unpaused, we need to manually kickstart the loop again.
        if (!this.paused) {
          this.loop(); 
        }
        // If we just paused, we don't do anything. The 'loop' function
        // checks 'this.paused' at the top and will stop itself automatically.
      },

      // starts input, music, and the loop
      start: function() {
        this.controls.connect();
        this.sounds.backgroundMusic();
        this.loop();
      },

      // stops the loop and pops up a win/lose alert
      stop: function(reason) {
        this.finished = true;
        window.clearTimeout(this.timer);
        alert(reason == "win" ? "You won!" : "You lost!");
      }
    };

    // restart button reloads the page
    document.getElementById("restartBtn").addEventListener("click", function () {
      location.reload();
    });

    // the world contains the level layout, physics, and collision map
    var world = {
      height: 480,
      width: 640,
      gravity: 10,
      distanceTravelled: 0,
      level: null, 
      collisionMap: null,
      tickCount: 0,
      enemies: [],

      loadLevel: function() {
        this.level = new Image();
        this.level.src = "Images/level.png";

        var collisionMapImage = new Image();
        collisionMapImage.onload = function(loadEvent) {
          var hiddenCanvas = document.createElement("CANVAS");
          hiddenCanvas.setAttribute("width", this.width);
          hiddenCanvas.setAttribute("height", this.height);
          world.collisionMap = hiddenCanvas.getContext("2d");
          world.collisionMap.drawImage(this, 0, 0);
        };
        // NOTE: You still need to insert your Base64 string below
        collisionMapImage.src = "data:image/png;base64, INSERT BASE64 COLLISION MAP HERE";
      },

      getFloorBelowY: function(x, y) {
        for (var tempY = y; tempY <= world.height; tempY++) {
          if (this.isSolidSurface(x, tempY)) {
            return tempY;
          }
        }
        return 0;
      },
      isSolidSurface: function(x, y) {
        return this.getPixelType(x, y) == "#";
      },
      getPixelType: function(x, y) {
        if (!this.collisionMap) {
          return ".";
        }
        var rawData = this.collisionMap.getImageData(x, y, 1, 1).data;
        var mask = rawData[0] + " " + rawData[1] + " " + rawData[2] + " " + rawData[3];
        if (mask == "255 0 0 255") return "pit";
        if (mask == "76 255 0 255") return "exit";
        if (mask == "255 255 255 255") return ".";
        if (mask == "0 0 0 255") return "#";
      },
      tick: function() {
        if (!this.level) {
          this.loadLevel();
          this.enemies.push(new enemy(500, 100));
          this.enemies.push(new enemy(2000, 100));
          this.enemies.push(new enemy(3700, 100));
          this.enemies.push(new enemy(4000, 100));
          this.enemies.push(new enemy(5600, 100));
          this.enemies.push(new enemy(6500, 100));
          this.enemies.push(new enemy(7600, 100));
        }
        this.distanceTravelled += player.character.speed;
        this.tickCount++;
        this.activateEnemies();
      },
      activateEnemies: function() {
        for (var i = 0; i < this.enemies.length; i++) {
          this.enemies[i].tick();
        }
      },
      levelEndOffset: function() {
        return this.level.width - this.width;
      },
      atLevelEnd: function() {
        return this.distanceTravelled >= this.levelEndOffset();
      },
      draw: function() {
        var drawAtX = this.distanceTravelled * -1;
        drawAtX = drawAtX > 0 ? 0 : drawAtX;
        drawAtX = this.atLevelEnd() ? this.levelEndOffset() * -1 : drawAtX;
        ctx.drawImage(this.level, drawAtX, 0);
        for (var i = 0; i < this.enemies.length; i++) {
          this.enemies[i].draw();
        }
      }
    };
    var player = {
      character: new character(160, 390, 25, 25, new animation("Images/graphics/cat", 5), new animation("Images/graphics/cat.backwards", 5)),
      tick: function() {
        var currentLocation = world.getPixelType(this.character.leadingEdge(), this.character.y);
        if (currentLocation == "exit" || currentLocation == "pit") {
          var state = currentLocation == "exit" ? "win" : "lose";
          game.stop(state);
          return;
        }
        this.processControls();
        this.character.tick();
      },
      // controls the cat when the WASD keys are pressed
      processControls: function() {
        if (game.controls.restart && !game.restartKeyWasDown) { 
          location.reload(); 
        } 
        game.restartKeyWasDown = game.controls.restart;

        // REMOVED: The pause check is now handled in game.controls.buttonPress
        // This prevents the issue where we can't unpause because the loop stopped.

        if (game.controls.right) {
          this.character.speed = 5;
        }
        if (game.controls.left) {
          this.character.speed = -5;
        }
        if (!game.controls.left && !game.controls.right) {
          this.character.speed = 0;
        }
        if (game.controls.up && this.character.standingOnAPlatform()) {
          this.character.downwardForce = -8;
          game.sounds.jump();
        }
      },
      draw: function() {
        this.character.draw();
      }
    };
    
    // Changes the sprite so the cat looks like it is running and jumping.
    function character(x, y, width, height, runningSprite, reverseSprite) {
      this.x = x;
      this.y = y;
      this.height = height;
      this.width = width;
      this.speed = 0;
      this.downwardForce = 0;
      this.jumpHeight = 0;
      this.runningSprite = runningSprite;
      this.runningSpriteReversed = reverseSprite;
      this.tick = function() {
        this.applyGravity();
        this.applyMovement();
      }

      // This code chunk makes the cat fall after it jumps
      this.applyGravity = function() {
        if (this.isJumping()) {
          this.jumpHeight += (this.downwardForce * -0.5);
          if (this.jumpHeight >= this.height * 4) {
            this.downwardForce = world.gravity;
            this.jumpHeight = 0;
          }
        } else {
          if (this.standingOnAPlatform()) {
            this.downwardForce = 0;
          } else {
            this.downwardForce = world.gravity;
          }
        }
      }
      this.applyMovement = function() {
        var nextX = this.x + this.speed;
        var nextY = this.y + this.downwardForce;
        var nextLeadingX = this.leadingEdge() + this.speed;
        var walkingIntoSurface = world.isSolidSurface(nextLeadingX, this.y);
        if (this.isMoving() && walkingIntoSurface) {
          nextX = this.x;
          this.speed = 0;
        }
        var topLeftIsSolid = world.isSolidSurface(this.leadingEdge(), this.y);
        var topRightIsSolid = world.isSolidSurface(this.trailingEdge(), this.y);
        if ((topLeftIsSolid || topRightIsSolid) && this.isJumping()) {
          this.downwardForce = world.gravity;
          this.jumpHeight = 0;
        }
        this.x = nextX;
        this.y = nextY;
      }
      this.bottom = function() {
        return this.y + this.height;
      }
      this.isJumping = function() {
        return this.downwardForce < 0;
      }
      this.isFalling = function() {
        return this.downwardForce > 0;
      }
      this.isMoving = function() {
        return this.speed != 0;
      }
      this.leadingEdge = function() {
        return this.speed < 0 ? this.x : this.x + this.width;
      }
      this.trailingEdge = function() {
        return this.speed < 0 ? this.x + this.width : this.x;
      }
      this.standingOnAPlatform = function() {
        return world.isSolidSurface(this.leadingEdge(), this.bottom() + 1) ||
          world.isSolidSurface(this.trailingEdge(), this.bottom() + 1);
      }
      this.draw = function() {
        if (!this.runningSprite) {
          return;
        }
        var drawAtX = this.x - world.distanceTravelled;
        drawAtX = drawAtX > this.x ? this.x : drawAtX;
        if (world.atLevelEnd()) {
          drawAtX = (world.width - (world.level.width - world.distanceTravelled - (this.x - world.distanceTravelled)));
        }
        var sprite = this.speed < 0 ? this.runningSpriteReversed : this.runningSprite;
        if (this.isJumping() || this.isFalling()) {
          sprite.drawFrame(4, drawAtX, this.y, this.height, this.width);
        } else if (this.isMoving()) {
          sprite.draw(world.tickCount, drawAtX, this.y, this.height, this.width);
        } else {
          sprite.drawFrame(1, drawAtX, this.y, this.height, this.width);
        }
      }
      this.collidesWith = function(other) {
        if (this.x >= other.x &&
          this.x <= other.x + other.width &&
          this.y >= other.y &&
          this.y <= other.y + other.height) {
          return true;
        }
        return false;
      }
    }

    function animation(filename, frameCount) {
      this.frames = [];
      this.currentFrameId = 1;
      for (var frameId = 1; frameId <= frameCount; frameId++) {
        var frame = new Image();
        frame.src = filename + "." + frameId + ".png";
        this.frames[frameId] = frame;
      }
      this.draw = function(tickCount, x, y, height, width) {
        if (tickCount % 5 == 0) {
          this.currentFrameId++;
        }
        this.currentFrameId = this.currentFrameId >= this.frames.length ? 1 : this.currentFrameId;
        this.drawFrame(this.currentFrameId, x, y, height, width);
      }
      this.drawFrame = function(frameNumber, x, y, height, width) {
        ctx.drawImage(this.frames[frameNumber], x, y, width, height);
      }
    }

    function enemy(x, y) {
      this.character = new character(x, y, 25, 25, new animation("Images/graphics/tiger", 5), new animation("Images/graphics/tiger.backwards", 5))
      this.tick = function() {
        var distanceFromPlayer = Math.abs(player.character.x - this.character.x);
        if (distanceFromPlayer <= world.width * 2) {
          this.character.speed = 2;
          if (player.character.x < this.character.x) {
            this.character.speed *= -1;
          }
          if (this.character.collidesWith(player.character)) {
            game.stop("lose");
          }
          this.character.tick();
        }
      }
      this.draw = function() {
        this.character.draw();
      }
    }
    game.start();
    document.getElementById("pauseBtn").addEventListener("click", function () {
      game.togglePause();
      this.textContent = game.paused ? "Resume" : "Pause";
    });
  </script>

```

### Explanation of the Changes

1. **Added Space Bar to `mapping`:**
I added `32: "pause"` to the controls mapping. `32` is the key code for Space. Now the game knows that space means "pause".
2. **Move Pause Logic to `buttonPress`:**
In your original code, the game waited for the `loop` to run `player.tick()`, which checked the controls. But since pausing stops the loop, `player.tick()` never ran, and you couldn't unpause!
I moved the check inside `buttonPress`. This runs immediately when you hit the key, even if the game loop is stopped.
3. **Fixed the Loop Logic:**
In `game.loop`, I added `if (this.paused) return;`.
In your old code, even when paused, the code would reach `window.setTimeout`, scheduling a "ghost" frame. When you clicked the button to unpause, you started a *second* loop alongside the ghost loop. Now, the loop completely stops scheduling itself when paused, preventing the double-speed bug.