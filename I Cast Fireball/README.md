# About
I Cast Fireball is anasyncronous online multiplayer game based on Rock-Paper-Scissors, with the addition of Fireball and Counterspell. Fireball wins against everything except for Counterspell, and Counterspell loses to everything except for Fireball.

 <img src="images\ICF_logo.png" alt="Logo for I Cast Fireball, diagram explaining the RPS rules in addition to the fireball and counterspell" width=75%>

The multiplayer aspect is achieved using the authorization and database cloudservices accessile via Firebase.

A user must create an account to play, and one must know the username of their desired opponent (no matchmaking), one can create as many games as one wishes and load in and play them asyncrounsly at any time (similar to Wordfeud).

Repository: [[Link]](https://github.com/LadyRonja/Monster-Dueler-Online-Async)

# My Responsibilities

## <b>Initializing the Database</b>
To abstract the accessibilty to the database within the project I created a singleton with the responsibility of ensuring the database is initialized and all dependencies are fixed.

<i>Click the dropdown arrow below to see the code</i>

<details>
<summary>FirebaseInitializer.cs</summary>
  
```csharp
using Firebase.Auth;
using Firebase;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Firebase.Extensions;
using Firebase.Database;

public class FirebaseInitializer : MonoBehaviour
{
    private static FirebaseInitializer instance;

    private FirebaseAuth _auth;
    private FirebaseDatabase _db;

    public static FirebaseInitializer Instance { get => _GetInstance(); private set => instance = value; }
    public static FirebaseAuth auth { get => _GetAuth(); private set => Instance._auth = value; }
    public static FirebaseDatabase db { get => _GetDB(); private set => Instance._db = value; }

    void Awake()
    {
        if (instance == null || instance == this)
            instance = this;
        else
            Destroy(this);

        FirebaseApp.CheckAndFixDependenciesAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);

            Instance._auth = FirebaseAuth.DefaultInstance;
            Instance._db = FirebaseDatabase.DefaultInstance;

            FirebaseDatabase.DefaultInstance.SetPersistenceEnabled(false);
        });
    }

    private static FirebaseInitializer _GetInstance()
    {
        if(instance != null)
            return instance;

        GameObject newInstance = new GameObject("Firebase Initializer");
        return newInstance.AddComponent<FirebaseInitializer>();
    }

    private static FirebaseAuth _GetAuth()
    {
        return Instance._auth;
    }

    private static FirebaseDatabase _GetDB()
    {
        return Instance._db;
    }
}
```
</details>


## <b>Saving to the Database</b>
A static class allowing saving generic data to a generic table in the database.

 <img src="images\database.png" alt="Screengrab from the database showing the table of a game between two players" width=40%>

<i>Click the dropdown arrow below to see the code</i>

<details>
<summary>FirebaseSaver.cs</summary>
  
```csharp

using Firebase.Auth;
using Firebase.Database;

public class FirebaseSaver
{
    public static async void SaveToDatabase(string saveToTable, string saveToPath, string jsonString)
    {
        var db = FirebaseInitializer.db;
        if (FirebaseInitializer.auth.CurrentUser == null)
            return;

        //puts the JSON data in the "saveToTable/saveToPath" part of the database.
        await db.RootReference.Child(saveToTable).Child(saveToPath).SetRawJsonValueAsync(jsonString);
    }

    public static async void SaveValueToDatabase(string saveToTable, string saveToPath, string value)
    {
        var db = FirebaseInitializer.db;
        if (FirebaseInitializer.auth.CurrentUser == null)
            return;

        //puts the JSON data in the "saveToTable/saveToPath" part of the database.
        await db.RootReference.Child(saveToTable).Child(saveToPath).SetValueAsync(value);
    }
}
```
</details>

## <b>Loading from the Database</b>
A static class meant to allow loading generic data from a generic table within the database.

Due to an issue with Unity's built in jsonmanager, some datatypes required their own return functions. 

<i>Click the dropdown arrow below to see the code</i>

<details>
<summary>FirebaseLoader.cs</summary>
  
```csharp
using Firebase.Auth;
using Firebase.Database;
using Firebase.Extensions;
using Google.MiniJSON;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;

public class FirebaseLoader
{
   public static async Task<string> LoadFromDatabase(string loadFromTable, string loadFromPath)
   {
        string output = "";
        var db = FirebaseInitializer.db;
        await db.RootReference.Child(loadFromTable).Child(loadFromPath).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
               Debug.LogError(task.Exception);
            }

            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            //And send the JSON data to a function that can update our game.
            output = snap.GetRawJsonValue();
        });
        return output;
   }

    public static async Task<string> LoadByValue(string loadFromTable, string basedOnValue, string value)
    {
        string output = "";
        var db = FirebaseInitializer.db;
        await db.RootReference.Child(loadFromTable).OrderByChild(basedOnValue).EqualTo(value).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
                Debug.LogError(task.Exception);
            }

            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            //And send the JSON data to a function that can update our game.
            output = snap.GetRawJsonValue();
        });
        return output;
    }

    public static async Task<User> GetUserFromUserNamer(string userName)
    {
        User output = null;
        var db = FirebaseInitializer.db;
        await db.RootReference.Child(DBPaths.USER_TABLE).OrderByChild("username").EqualTo(userName).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
                Debug.LogError(task.Exception);
            }
            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            if (snap.ChildrenCount > 0)
            {
                foreach (var item in snap.Children)
                {
                    string json = item.GetRawJsonValue();
                    output = JsonUtility.FromJson<User>(json);
                }
            }
        });
        return output;
    }

    public static async Task<List<RPSGame>> GetGamesWithUser(User withUser, bool rpsGame = false)
    {
        List<RPSGame> output = new();
        var db = FirebaseInitializer.db;

        string tableToLoad = DBPaths.GAMES_TABLE;
        if(rpsGame) tableToLoad = DBPaths.RPS_TABLE;
        await db.RootReference.Child(tableToLoad).OrderByChild("playerA").EqualTo(withUser.username).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
                Debug.LogError(task.Exception);
            }
            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            if (snap.ChildrenCount > 0)
            {
                foreach (var item in snap.Children)
                {
                    string json = item.GetRawJsonValue();
                    output.Add(JsonUtility.FromJson<RPSGame>(json));
                }
            }
        });

        await db.RootReference.Child(tableToLoad).OrderByChild("playerB").EqualTo(withUser.username).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
                Debug.LogError(task.Exception);
            }
            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            if (snap.ChildrenCount > 0)
            {
                foreach (var item in snap.Children)
                {
                    string json = item.GetRawJsonValue();
                    output.Add(JsonUtility.FromJson<RPSGame>(json));
                }
            }
        });
        return output;
    }

    public static async Task<string> LoadTable(string tableName)
    {
        string output = "";

        var db = FirebaseInitializer.db;
        await db.GetReference(tableName).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
            {
                Debug.LogError(task.Exception);
            }
            //here we get the result from our database.
            DataSnapshot snap = task.Result;

            //And send the JSON data to a function that can update our game.
            output = snap.GetRawJsonValue();
            return output;
        });
        return output;
    }
}
```
</details>


## <b>Loading game instances</b>
I wrote this class to load in the gamestate whenever a change happens on the database if one is currently viewing the game that was updated.

This also loads a game whenever oen is opened up as well.

This class does have too much responsibility as to load the gamestate properly just based on the minimum data stored in the database, it also needs to know the logic for what move wins over what other move. 

This should be split into another class.

The rules are hardcoded in as a list of touples that compares against itself to find a the reult for each move the players has made.

 <img src="images\gameplay.gif" alt="Gameplay video showing the creation of a new game, some moves, and loading back into it" width=75%>

<i>Click the dropdown arrow below to see the code</i>

<details>
<summary>RPSLoader.cs</summary>
  
```csharp
using AYellowpaper.SerializedCollections;
using Firebase.Database;
using System;
using System.Collections;
using System.Collections.Generic;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public class RPSLoader : MonoBehaviour
{
    // Singleton
    private static RPSLoader instance;
    public static RPSLoader Instance { get => GetInstance(); private set => instance = value; }

    public static string rpsGameToLoad = "";
    public const int MAX_MOVES = 10;
    // Development purpose
    public string debugGame = "339fd4bf-c9be-4468-ab2c-304f41516327";
    public bool usingDebug = true;
    [Space(20)]
    [SerializeField] GameObject moveEntryPrefab;
    [SerializedDictionary("ID, Card")]
    public SerializedDictionary<RPS, Sprite> moveToSpriteDictionary;
    [SerializeField] Transform activePlayerEntryParent;
    [SerializeField] Transform opponentPlayerEntryParent;
    [Space]
    [SerializeField] TMP_Text activePlayerNameText;
    [SerializeField] TMP_Text opponentNameText;
    [Space]
    [SerializeField] TMP_Text winsText;
    [SerializeField] TMP_Text tiesText;
    [SerializeField] TMP_Text lossesText;
    [Space]
    [SerializeField] TMP_Text victoryText;

    [HideInInspector]
    public RPSGame loadedGame;
    public List<RPSMove> activePlayerMoves;
    public List<RPSMove> opponentMoves;

    public delegate void LoadedGame();
    public LoadedGame OnStateUpdated;
    public LoadedGame OnGameLoaded;

    enum MoveResult {UNDETERMINED, WON, TIED, LOST }
    List<(RPS myMove, List<RPS> opponentsMove, MoveResult result)> rules = new();

    private void Awake()
    {
        #region Singleton
        if (instance == null || instance == this)
            instance = this;
        else
            Destroy(this.gameObject);
        #endregion

        Debug.Log("should load: " + rpsGameToLoad);
        Debug.Log("devgame id: " + debugGame);
        foreach (Transform child in activePlayerEntryParent)
            Destroy(child.gameObject);
        foreach (Transform child in opponentPlayerEntryParent)
            Destroy(child.gameObject);

        CreateRules();

        Invoke(nameof(FetchGame), 1);
        Invoke(nameof(Subscribe), 1);
    }

    private void CreateRules()
    {
        rules = new();

        // Soldier
        rules.Add((RPS.SOLDIER,
                    new() {
                        RPS.POLITICIAN, 
                        RPS.FIREBALL},
                            MoveResult.LOST));

        rules.Add((RPS.SOLDIER,
            new() {
                        RPS.ASSASSIN,
                        RPS.COUNTER_SPELL},
                    MoveResult.WON));

        // Assassin
        rules.Add((RPS.ASSASSIN,
            new() {
                        RPS.SOLDIER,
                        RPS.FIREBALL},
                    MoveResult.LOST));

        rules.Add((RPS.ASSASSIN,
            new() {
                        RPS.POLITICIAN,
                        RPS.COUNTER_SPELL},
                    MoveResult.WON));

        // Politician
        rules.Add((RPS.POLITICIAN,
            new() {
                        RPS.ASSASSIN,
                        RPS.FIREBALL},
                    MoveResult.LOST));

        rules.Add((RPS.POLITICIAN,
            new() {
                        RPS.SOLDIER,
                        RPS.COUNTER_SPELL},
                    MoveResult.WON));

        // Fireball
        rules.Add((RPS.FIREBALL,
            new() {
                        RPS.ASSASSIN,
                        RPS.POLITICIAN,
                        RPS.SOLDIER},
                    MoveResult.WON));

        rules.Add((RPS.FIREBALL,
            new() {
                        RPS.COUNTER_SPELL},
                    MoveResult.LOST));

        // Counter Spell
        rules.Add((RPS.COUNTER_SPELL,
            new() {
                        RPS.FIREBALL},
                    MoveResult.WON));

        rules.Add((RPS.COUNTER_SPELL,
            new() {
                        RPS.ASSASSIN,
                        RPS.POLITICIAN,
                        RPS.SOLDIER},
                    MoveResult.LOST));

    }
    private void Subscribe()
    {
        string idToTack = rpsGameToLoad;
        if (usingDebug) idToTack = debugGame;

        FirebaseInitializer.db.GetReference($"{DBPaths.RPS_TABLE}/{idToTack}").ValueChanged += HandleValueChange;
    }

    private void HandleValueChange(object sender, ValueChangedEventArgs args)
    {
        if (args.DatabaseError != null)
        {
            //Debug.LogError(args.DatabaseError.Message);
            return;
        }

        // Do something with the data in args.Snapshot
        Debug.Log("Value has changed: " + args.Snapshot.GetRawJsonValue());

        //run the game with the new information
        FetchGame();
    }


    public async void FetchGame()
    {
        if (rpsGameToLoad == "" && !usingDebug)
        {
            Debug.LogError("No game to load");
            return;
        }

        string gameBlob;
        if (usingDebug)
        {
            gameBlob = await FirebaseLoader.LoadFromDatabase(DBPaths.RPS_TABLE, debugGame);
            string activeUserBlob = await FirebaseLoader.LoadFromDatabase(DBPaths.USER_TABLE, "9GZbdeUZ5PPlkkmZAJMgJPvKoDj2");
            ActiveUser.SetActiveUser(JsonUtility.FromJson<User>(activeUserBlob));
        }
        else
            gameBlob = await FirebaseLoader.LoadFromDatabase(DBPaths.RPS_TABLE, rpsGameToLoad);


        if (gameBlob == null || gameBlob == "")
        {

            Debug.LogError("Game not found");
            return;
        }


        RPSGame gameToLoad = JsonUtility.FromJson<RPSGame>(gameBlob);

        // TODO:
        // Validate game state here

        LoadGame(gameToLoad);
    }

    private void LoadGame(RPSGame gameToLoad)
    {
        Debug.Log($"Signed in player: {ActiveUser.CurrentActiveUser.username}");

        foreach (Transform child in activePlayerEntryParent)
            Destroy(child.gameObject);
        foreach (Transform child in opponentPlayerEntryParent)
            Destroy(child.gameObject);


        if (usingDebug)
            Debug.Log("Loading game with current information: " + debugGame);
        else
            Debug.Log("Loading game with current information: " + rpsGameToLoad);


        int movesToLoad = Mathf.Min(gameToLoad.playerAMoves.Count, gameToLoad.playerBMoves.Count); 
        activePlayerMoves = new();
        opponentMoves = new();

        if (gameToLoad.playerA == ActiveUser.CurrentActiveUser.username)
        {
            activePlayerMoves = gameToLoad.playerAMoves;
            opponentMoves = gameToLoad.playerBMoves;
            activePlayerNameText.text = gameToLoad.playerA;
            opponentNameText.text = gameToLoad.playerB;
        }
        else
        {
            activePlayerMoves = gameToLoad.playerBMoves;
            opponentMoves = gameToLoad.playerAMoves;
            activePlayerNameText.text = gameToLoad.playerB;
            opponentNameText.text = gameToLoad.playerA;
        }

        int wins = 0;
        int ties = 0;
        int losses = 0;
        for (int i = 0; i < movesToLoad; i++)
        {
            GameObject aMove = Instantiate(moveEntryPrefab, activePlayerEntryParent);
            Image aImage = aMove.GetComponentInChildren<Image>();
            aImage.sprite = moveToSpriteDictionary[activePlayerMoves[i].selectedMove];
            if (activePlayerMoves[i].selectedMove != RPS.FIREBALL && activePlayerMoves[i].selectedMove != RPS.COUNTER_SPELL)
                aImage.color = Color.black;

            GameObject oMove = Instantiate(moveEntryPrefab, opponentPlayerEntryParent);
            Image oImage = oMove.GetComponentInChildren<Image>();
            oImage.sprite = moveToSpriteDictionary[opponentMoves[i].selectedMove];
            if (opponentMoves[i].selectedMove != RPS.FIREBALL && opponentMoves[i].selectedMove != RPS.COUNTER_SPELL)
                oImage.color = Color.black;

            MoveResult result = MoveResult.UNDETERMINED;

            foreach (var rule in rules)
            {
                if (activePlayerMoves[i].selectedMove != rule.myMove)
                    continue;

                if (rule.opponentsMove.Contains(opponentMoves[i].selectedMove))
                {
                    result = rule.result; 
                    break;
                }
            }

            switch (result)
            {
                case MoveResult.WON:
                    wins++;
                    aMove.GetComponentsInChildren<Image>()[1].color = Color.green;
                    oMove.GetComponentsInChildren<Image>()[1].color = Color.red;
                    break;
                case MoveResult.LOST:
                    losses++;
                    aMove.GetComponentsInChildren<Image>()[1].color = Color.red;
                    oMove.GetComponentsInChildren<Image>()[1].color = Color.green;
                    break;
                case MoveResult.UNDETERMINED:
                case MoveResult.TIED:
                    ties++;
                    aMove.GetComponentsInChildren<Image>()[1].color = Color.yellow;
                    oMove.GetComponentsInChildren<Image>()[1].color = Color.yellow;
                    break;
            }
        }

        if (activePlayerMoves.Count == opponentMoves.Count+1) 
        {
            GameObject aMove = Instantiate(moveEntryPrefab, activePlayerEntryParent);
            Image aImage = aMove.GetComponentInChildren<Image>();
            aImage.sprite = moveToSpriteDictionary[activePlayerMoves[^1].selectedMove];
            if (activePlayerMoves[^1].selectedMove != RPS.FIREBALL && activePlayerMoves[^1].selectedMove != RPS.COUNTER_SPELL)
                aImage.color = Color.black;
        }

        winsText.color = Color.green;
        winsText.text = $"Wins: {wins}";

        tiesText.color = Color.yellow;
        tiesText.text = $"Ties: {ties}";

        lossesText.color = Color.red;
        lossesText.text = $"Lost: {losses}";

        loadedGame = gameToLoad;

        if (IsGameOver(movesToLoad))
        {
            MoveResult winState = MoveResult.TIED;
            if (wins > losses) winState = MoveResult.WON;
            else if (wins < losses) winState = MoveResult.LOST;
            CompleteGame(gameToLoad, winState);
        }

        OnGameLoaded?.Invoke();

    }

    private bool IsGameOver(int movesToLoad)
    {
        if (movesToLoad >= MAX_MOVES)
            return true;
        else 
            return false;
    }

    private void CompleteGame(RPSGame gameToComplete, MoveResult activePlayerWon)
    {
        gameToComplete.gameIsOver = true;
        if(gameToComplete.gameDoneAt == 0) gameToComplete.gameDoneAt = DateTime.Now.Ticks;
        string gameBlob = JsonUtility.ToJson(gameToComplete);
        FirebaseSaver.SaveToDatabase(DBPaths.RPS_TABLE, gameToComplete.gameID, gameBlob);
        loadedGame = gameToComplete;

        switch (activePlayerWon)
        {
            case MoveResult.WON:
                victoryText.text = "YOU WON!";
                gameToComplete.winnner = ActiveUser.CurrentActiveUser.username;
                victoryText.color = Color.green;
                break;
            case MoveResult.TIED:
                victoryText.text = "TIE";
                gameToComplete.winnner = "";
                victoryText.color = Color.yellow;
                break;
            case MoveResult.LOST:
                victoryText.text = "YOU LOST!";
                if (ActiveUser.CurrentActiveUser.username.Equals(gameToComplete.playerA)) gameToComplete.winnner = gameToComplete.playerB;
                else gameToComplete.winnner = gameToComplete.playerA;
                victoryText.color = Color.red;
                break;
        }

        victoryText.gameObject.SetActive(true);
    }

    private void OnDisable()
    {
        OnDestroy();
    }

    private void OnDestroy()
    {
        string idToTack = rpsGameToLoad;
        if (usingDebug) idToTack = debugGame;

        FirebaseInitializer.db.GetReference($"{DBPaths.RPS_TABLE}/{idToTack}").ValueChanged -= HandleValueChange;
    }

    private static RPSLoader GetInstance()
    {
        if (instance != null)
            return instance;

        instance = new GameObject("GameLoader Manager").AddComponent<RPSLoader>();
        return instance;
    }
}
```
</details>
