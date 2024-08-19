# About
I Cast Fireball is anasyncronous online multiplayer game based on Rock-Paper-Scissors, with the addition of Fireball and Counterspell. Fireball wins against everything except for Counterspell, and Counterspell loses to everything except for Fireball.
The multiplayer aspect is achieved using the authorization and database cloudservices accessile via Firebase.
A user must create an account to play, and one must know the username of their desired opponent (no matchmaking), one can create as many games as one wishes and load in and play them asyncrounsly at any time (similar to Wordfeud).

Repository: [Link]

# My Responsibilities

<b>FirebaseInitializer</b>
To abstract the accessibilty to the database within the project I created a singleton with the responsibility of ensuring the database is initialized and all dependencies are fixed.
[Code Foldout]

<b>FirebaseSaver</b>
A static class allowing saving generic data to a generic table in the database.
[Code Foldout]

<b>FirebaseLoader</b>
A static class meant to allow loading generic data from a generic table within the database.
Due to an issue with Unity's built in jsonmanager, some datatypes required their own return functions. 
[Code Foldout]


<b>RPSLoader</b>
I wrote this class to load in the gamestate whenever a change happens on the database if one is currently viewing the game that was updated.
This also loads a game whenever oen is opened up as well.
This class does have too much responsibility as to load the gamestate properly just based on the minimum data stored in the database, it also needs to know the logic for what move wins over what other move. This should be split into another class.
The rules are hardcoded in as a list of touples that compares against itself to find a the reult for each move the players has made.
[Code Foldout]
