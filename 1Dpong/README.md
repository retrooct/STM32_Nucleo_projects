# STM32 1D LED Pong Game

## Project Overview

This project is a two-player, one-dimensional Pong game implemented in C on an STM32 Nucleo microcontroller.

Instead of displaying the game on a traditional screen, the ball is represented by a row of LEDs. The ball moves from one side of the LED array to the other, and each player must press a button when the ball reaches their side.

The project demonstrates several embedded-system concepts, including:

* GPIO input and output
* Direct register manipulation
* Hardware interrupts
* Software timing
* Button input and debouncing
* Finite-state machine design
* Real-time game logic

---

## What Does the Game Do?

The game uses eight LEDs to represent the playing field.

A single illuminated LED represents the ball. The ball shifts across the LED array toward one of the players.

When the ball reaches a player's side:

1. The player must press their button within the correct timing window.
2. A successful hit sends the ball toward the other player.
3. A missed hit awards a point to the opposing player.
4. The first player to reach three points wins the game.

After a player wins, the score LEDs flash before the game resets.

---

## Hardware Used

* STM32L476 Nucleo development board
* Eight LEDs for the playing field
* Six LEDs for player scores
* Two player push buttons
* STM32 Nucleo user button
* Current-limiting resistors
* Breadboard
* Jumper wires

---

## Software and Tools

* C
* STM32CubeIDE
* STM32 HAL and CMSIS register definitions
* SysTick timer
* Timer 2
* GPIO registers
* Git and GitHub

---

## GPIO Pin Configuration

### Playing Field LEDs

The eight playing-field LEDs are connected to GPIO Port C.

| Function    | STM32 Pin |
| ----------- | --------- |
| Field LED 1 | PC5       |
| Field LED 2 | PC6       |
| Field LED 3 | PC7       |
| Field LED 4 | PC8       |
| Field LED 5 | PC9       |
| Field LED 6 | PC10      |
| Field LED 7 | PC11      |
| Field LED 8 | PC12      |

### Player Input Buttons

| Function            | STM32 Pin |
| ------------------- | --------- |
| Left player button  | PC1       |
| Right player button | PC0       |
| Mode-control button | PC13      |

### Score LEDs

| Function       | STM32 Pins      |
| -------------- | --------------- |
| Player 1 score | PC14, PC15, PH0 |
| Player 2 score | PH1, PC2, PC3   |

---

## My Hardware Design

### Controlling the LED Array Through GPIO Port C

The playing-field LEDs are connected to pins PC5 through PC12. Because the LEDs occupy consecutive GPIO pins, their states can be controlled as a single eight-bit pattern.

The game stores the current position of the ball in an eight-bit variable.

Example:

```c
uint8_t ledPattern = 0x01;
```

Each bit represents one LED:

```text
00000001
```

As the ball moves, the pattern is shifted left or right:

```c
ledPattern <<= 1;
```

or:

```c
ledPattern >>= 1;
```

The pattern is then shifted into the correct GPIO Port C bit positions and written to the GPIO output data register.

```c
GPIOC->ODR &= ~(0xFFU << 5);
GPIOC->ODR |= ((uint32_t)ledPattern << 5);
```

The first line clears pins PC5 through PC12.

The second line writes the current LED pattern into those pins.

This approach allows all eight playing-field LEDs to be updated with a single register operation instead of controlling every LED separately.

---

## Software Design

The game is organized as a finite-state machine.

Each state represents a specific stage of gameplay.

### Main Game States

```text
STATE_SERVE
STATE_SHIFT_LEFT
STATE_SHIFT_RIGHT
STATE_CHECK_LEFT_HIT
STATE_CHECK_RIGHT_HIT
STATE_WIN
```

### State Descriptions

#### `STATE_SERVE`

Places the ball at one side of the field and waits for the round to begin.

#### `STATE_SHIFT_LEFT`

Moves the ball one LED position toward the left player.

#### `STATE_SHIFT_RIGHT`

Moves the ball one LED position toward the right player.

#### `STATE_CHECK_LEFT_HIT`

Checks whether the left player pressed the button while the ball was within the valid hit position.

#### `STATE_CHECK_RIGHT_HIT`

Checks whether the right player pressed the button while the ball was within the valid hit position.

#### `STATE_WIN`

Displays the winner and flashes the player's score LEDs before resetting the game.

---

## Example State Flow

```text
Serve
  |
  v
Shift Right
  |
  v
Check Right Hit
  |
  +------ Successful Hit ------> Shift Left
  |
  +------ Missed Hit ----------> Award Point
                                  |
                                  v
                              Next Serve
```

The same process occurs in the opposite direction for the other player.

---

## Ball Movement

The ball is represented by one active bit inside the LED pattern.

For example:

```text
00000001
00000010
00000100
00001000
00010000
00100000
01000000
10000000
```

A left or right bit shift changes the active LED and creates the appearance that the ball is moving across the field.

Example movement functions:

```c
uint8_t moveRight(uint8_t pattern)
{
    if (pattern == 0x80)
    {
        return 0;
    }

    return pattern << 1;
}
```

```c
uint8_t moveLeft(uint8_t pattern)
{
    if (pattern == 0x01)
    {
        return 0;
    }

    return pattern >> 1;
}
```

Returning zero indicates that the ball has moved past the end of the playing field.

---

## Timing Design

### SysTick

SysTick provides a regular timing reference for game events.

It is used to control:

* LED movement speed
* Score LED flashing
* Game-mode timing
* Periodic state-machine updates

The timer allows the game to continue operating without relying entirely on long blocking delays.

### Timer 2

Timer 2 is used for button handling and timing-related input processing.

It can be used to:

* sample player buttons
* debounce mechanical button input
* detect valid button presses
* prevent one physical press from registering multiple times

Separating input timing from the primary game logic helps make the controls more responsive and predictable.

---

## Button Debouncing

Mechanical buttons do not immediately transition cleanly between pressed and released states. A single press may create several rapid electrical transitions.

Without debouncing, the program may interpret one press as multiple presses.

The project addresses this by sampling button input over time and accepting the input only after it remains stable for a defined interval.

Example logic:

```text
Read button
     |
     v
Has the state changed?
     |
    Yes
     |
     v
Start debounce counter
     |
     v
Has the state remained stable?
     |
    Yes
     |
     v
Register valid button press
```

---

## Scoring System

Each player has three score LEDs.

When a player earns a point, another score LED is illuminated.

```text
0 points: 000
1 point: 001
2 points: 011
3 points: 111
```

The first player to reach three points wins.

After detecting a winner, the game enters `STATE_WIN`, flashes the winner's score LEDs, and resets both scores.

---

## Serve Logic

The serve alternates between players instead of always beginning from the same side.

Example:

```c
if (currentServer == PLAYER_ONE)
{
    ledPattern = 0x01;
    currentServer = PLAYER_TWO;
}
else
{
    ledPattern = 0x80;
    currentServer = PLAYER_ONE;
}
```

This gives both players an equal opportunity to begin a round.

---

## Game Modes

The project includes separate operating modes.

### Play Mode

The normal game mode in which the ball moves, players press buttons, and scores are tracked.

### Flash LED Mode

A separate mode used to flash LEDs through SysTick-based timing.

The STM32 Nucleo user button on PC13 is used to switch between modes.

---

## Major Challenges

### 1. Updating Multiple LEDs Efficiently

Controlling each LED with separate instructions would make the code repetitive.

I addressed this by representing the LED array as an eight-bit value and writing the entire pattern to GPIO Port C.

---

### 2. Mapping the LED Pattern to PC5 Through PC12

The LED pattern begins at bit zero, but the physical LEDs begin at PC5.

The pattern therefore had to be shifted left by five positions before being written to the output data register.

```c
GPIOC->ODR |= ((uint32_t)ledPattern << 5);
```

---

### 3. Detecting Hits at the Correct Time

A button press should only count when the ball is near the player's side.

I used dedicated hit-check states to separate ball movement from input validation.

This prevented button presses made at unrelated times from being counted as valid hits.

---

### 4. Preventing Multiple Button Registrations

Mechanical button bounce could cause one press to register more than once.

Timer-based debouncing was used to verify that the input remained stable before accepting it.

---

### 5. Coordinating Game Timing and Input

The game needed to move the ball at a predictable speed while still responding to button input.

Timer interrupts and a finite-state machine allowed timing, movement, and input processing to be managed separately.

---

### 6. Resetting the Game Correctly

After a player won, the following values needed to be restored:

* Player scores
* Ball position
* Current state
* Serve direction
* LED pattern
* Timing variables

Resetting every required variable prevented the previous game from affecting the next game.

---

## What I Learned

Through this project, I gained experience with:

* Configuring STM32 GPIO pins
* Reading and writing GPIO registers
* Using masks and bit shifting
* Representing hardware states with binary values
* Creating finite-state machines
* Handling hardware timers and interrupts
* Debouncing mechanical buttons
* Organizing a real-time embedded C program
* Debugging hardware and software interactions
* Separating game logic from hardware-control functions

---

## Possible Future Improvements

* Add adjustable game-speed levels
* Increase ball speed after successful hits
* Add sound effects using a buzzer
* Use external interrupts for player buttons
* Display scores on a seven-segment display
* Add difficulty settings
* Create a printed circuit board for the complete game
* Add a reaction-time measurement mode
* Use PWM to change LED brightness
* Add a start menu using an OLED display

---

## Project Structure

```text
STM32-1D-Pong/
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── game.h
│   │   ├── gpio_control.h
│   │   └── button.h
│   └── Src/
│       ├── main.c
│       ├── game.c
│       ├── gpio_control.c
│       └── button.c
├── Drivers/
├── README.md
└── STM32-1D-Pong.ioc
```

Adjust the file structure above to match the actual organization of the project.

---

## Building and Running the Project

1. Open the project in STM32CubeIDE.
2. Connect the STM32 Nucleo board to the computer.
3. Build the project.
4. Flash the program to the board.
5. Connect the LED array and player buttons to their assigned GPIO pins.
6. Start the game and use the left and right player buttons to return the ball.

---

## Demonstration

Add images or videos of the completed system here.

```markdown
![STM32 1D Pong Hardware](images/stm32-pong-hardware.jpg)
```

```markdown
[Watch the demonstration video](YOUR_VIDEO_LINK)
```

---

## Author

**Humza Rana**

Computer Engineering
Kennesaw State University

GitHub: [HumzaProfessional](https://github.com/HumzaProfessional)
