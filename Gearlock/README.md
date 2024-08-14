# About
Gearlock is a singleplayer tactical push-your-luck game with deckbuilding elements.
As the player, your goal is to beat a level by defeating all enemies in the level. 
You will be using card actions to make robots do things like fighting, throwing explosives, and digging out new cards

Repository: [Link]
Itch: [Link]

# My Responsibilities
<b>Pathfinding </b> 
The pathfinding system is at its core A* with additional options such as "can walk through walls", "can walk through characthers", and "path must be X amount of tiles away or shorter."
[Code Foldout]

<b>Card Mechanical structure </b> 
Once a player has selected a card they wish to play, some variables must be selected by the player and then verified if they are legal.
If I were to refactor this I'd have each verification be done as the result of an event rather then checked each frame with a finite-state machine, however the structure is still solid even if the form can be optimized.
Once a player is attempting to play a card, the have to select which robot is going to use the card. Once a player makes a choice, we check if the choice is legal, if it is we move on selecting where the card is to be used on the map. If it's illegal the player has to chose another robot.
Repeat the above process to check if where they wish to play the card is legal, often this is based on the length away from the robot using the card.
Once fully legal choices has been made code associated with the specific card is executed, being sent the relevant data of the choices.
[Card flowchart Image here]
[Code Foldout]

- I also worked with
AI Behaivour 
Audio Manager
Camera Manager


