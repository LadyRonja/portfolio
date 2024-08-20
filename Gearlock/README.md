# About

 <img src="images\gearlock_logo.png" alt="Gearlock logo, a wooden sign with the name in metalic lettering" width=50%>
 
Gearlock is a singleplayer tactical push-your-luck game with deckbuilding elements.
As the player, your goal is to beat a level by defeating all enemies in the level. 
You will be using card actions to make robots do things like fighting, throwing explosives, and digging out new cards

Repository: [<a href="https://github.com/LadyRonja/Gearlock">Link</a>]

Itch: [<a href="https://yrgo-game-creator.itch.io/gearlock">Link</a>]


# My Responsibilities

## <b>Pathfinding </b> 

The pathfinding system is at its core A* with additional options such as "can walk through walls", "can walk through characthers", and "path must be X amount of tiles away or shorter."

 <img src="images\pathfinding.gif" alt="Gif showing a functional A* algorithm in a hex based grid" width=100%>

<i>Click the dropdown below for the full script.</i>

<details>
<summary>Pathfinding.cs</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.InteropServices.WindowsRuntime;
using Unity.Mathematics;
using UnityEngine;

public class Pathfinding : MonoBehaviour
{
    public static List<Tile> FindPath(Tile start, Tile end, int range, bool ignoreBlocks, bool ignoreWalls)
    {
        // Keep list of tiles to search and tiles already searched
        List<Tile> toSearch = new List<Tile>() { start };
        List<Tile> processed = new List<Tile>();

        // Keep going as long as we have any tiles to search
        while (toSearch.Any())
        {
            Tile current = toSearch[0];
            foreach (Tile t in toSearch)
            {
                // The current tile is the tile with the
                // lowest sum of
                // tiles away from the start tile,
                // and the lowest possible amount of tiles from the end
                if (t.F < current.F || t.F == current.F && t.H < current.H)
                {
                    current = t;
                }
            }

            processed.Add(current);
            toSearch.Remove(current);

            // End is reached, return the path that lead here
            if (current == end)
            {
                Tile currentPathTile = end;
                List<Tile> path = new List<Tile>();
                while (currentPathTile != start)
                {
                    path.Add(currentPathTile);
                    currentPathTile = currentPathTile.cameFrom;
                }
                path.Reverse();

                // AI will look for the tile the target is standing on, remove that
                if(path.Count != 0)
                {
                    if (path[^1].occupied)
                        path.RemoveAt(path.Count - 1);
                    else if (path[^1].blocked && !ignoreBlocks)
                        path.RemoveAt(path.Count - 1);
                }

                if(range >= 0)
                {
                    while(path.Count > range)
                    {
                        path.RemoveAt(path.Count - 1);
                    }
                }

                return path;
            }

            // Determine which neighbours to investigate
            List<Tile> investigateNeighbours;
            if (ignoreBlocks)
            {
                if(ignoreWalls)
                    investigateNeighbours = current.neighbours.Where(t => !t.occupied && !processed.Contains(t)).ToList();
                else
                    investigateNeighbours = current.neighbours.Where(t => !t.occupied && !processed.Contains(t) && !t.walled).ToList();

                // AI will look for a path where a unit is standing, make sure to find that tile
                foreach (Tile t in current.neighbours)
                {
                    if(t == end && !investigateNeighbours.Contains(t))
                    {
                        investigateNeighbours.Add(t);
                    }
                }
            }
            else
            {
                if(ignoreWalls)
                    investigateNeighbours = current.neighbours.Where(t => !t.occupied && !processed.Contains(t) && !t.blocked).ToList();
                else
                    investigateNeighbours = current.neighbours.Where(t => !t.occupied && !processed.Contains(t) && !t.blocked && !t.walled).ToList();

                // AI will look for a path where a unit is standing, make sure to find that tile
                foreach (Tile t in current.neighbours)
                {
                    if (t == end && !investigateNeighbours.Contains(t))
                    {
                        investigateNeighbours.Add(t);
                    }
                }
            }

            // Check each non-blocked neighbouring tile that has not already been processed
            foreach (Tile neighbour in investigateNeighbours)
            {
                // Prephare to set cost of neighbour
                bool inSearch = toSearch.Contains(neighbour);
                int costToNeighbour = current.G + GetDistance(current, neighbour);

                // If neighbour hasn't already been searched, or a new path has determined a lower cost than a previous one
                // Update connection and cost
                if (!inSearch || costToNeighbour < neighbour.G)
                {
                    neighbour.G = costToNeighbour;
                    neighbour.cameFrom = current;

                    // If not previously searched, determine heuristic value
                    if (!inSearch)
                    {
                        neighbour.H = GetDistance(neighbour, end);
                        toSearch.Add(neighbour);
                    }
                }
            }
        }

        // No possible path, return null
        return null;
    }

    public static List<Tile> FindPath(Tile start, Tile end)
    {
        return FindPath(start, end, -1, false, false);
    }

    public static List<Tile> FindPath(Tile start, Tile end, int range)
    {
        return FindPath(start, end, range, false, false);
    }

    public static int GetDistance(Tile a, Tile b)
    {
        int xDist = math.abs(a.x - b.x);
        int yDist = math.abs(a.y - b.y);
        int distTot = xDist + yDist;

        return distTot;

        #region Decripit
        // Diagonal move allowed
        /*Vector2 vec = new Vector2(a.x - b.x, a.y - b.y);
        return Mathf.CeilToInt(Mathf.Max(math.abs(vec.x), math.abs(vec.y)));*/
        #endregion
    }
}
```
</details>


## <b>Card Mechanical structure </b> 

Once a player has selected a card they wish to play, some variables must be selected by the player and then verified if they are legal.

If I were to refactor this I'd have each verification be done as the result of an event rather then checked each frame with a finite-state machine, however the structure is still solid even if the form can be optimized.

Once a player is attempting to play a card, the have to select which robot is going to use the card. Once a player makes a choice, we check if the choice is legal, if it is we move on selecting where the card is to be used on the map. If it's illegal the player has to chose another robot.

Repeat the above process to check if where they wish to play the card is legal, often this is based on the length away from the robot using the card.

Once fully legal choices has been made code associated with the specific card is executed, being sent the relevant data of the choices.

 <img src="images\CardLogicFlowchart.png" alt="Flowchart depicting the logic flow from selecting to using a card" width=30%>

<i>Click the dropdown below for the full script.</i>

<details>
<summary>Card.cs</summary>
  

 ```csharp
[using System.Collections.Generic;
using System.Linq;
using TMPro;
using Unity.VisualScripting;
//using UnityEditor.PackageManager;
using UnityEngine.EventSystems;
using UnityEngine;
using UnityEngine.UI;

public enum CardState
{
    Inactive,
    VerifyUnitSelection,
    SelectingUnit,
    SelectingTile,
    VerifyTileSelection,
    Executing,
    Finished
}

public abstract class Card : MonoBehaviour
{
    public enum CardType
    {
        Dig,
        Attack,
        Attack2x,
        DiggerBot,
        FighterBot,
        Dynamite
    }
    [Header("Info displayed to player")]
    public string cardName = "--";
    public Sprite cardFrame;
    public Color frameColor = Color.white;
    public Sprite illustration;
    [TextArea(6, 6)]
    public string cardDescription;
    public int range = 1;

    [Header("Card restrictions")]
    public BotSpecialization requiredSpecialization = BotSpecialization.None;
    public bool hasToTargetDirtTiles = true;
    public bool canNotTargetDirtTiles = false;
    public bool hasToTargetOccupiedTiles = true;
    public bool canNotTargetOccupiedTiles = false;
    public bool canNotTargetWalls = true;
    public bool goesToDiscardAfterPlay = true;

    protected bool unitsHighligthed = false;
    protected bool tilesHighligthed = false;
    protected bool cardExecutionCalled = false;

    [HideInInspector] public CardState myState = CardState.Inactive;
    public CardType myType;
    [HideInInspector] public Tile selectedTile = null;
    [HideInInspector] public Unit selectedUnit = null;

    protected virtual void Start()
    {
        myState = CardState.Inactive;

        if (canNotTargetOccupiedTiles && hasToTargetOccupiedTiles)
            Debug.LogError($"WARNING: {cardName} both has to and is unable to target occupied Tiles!");
        if (canNotTargetDirtTiles && hasToTargetDirtTiles)
            Debug.LogError($"WARNING: {cardName} both has to and is unable to target dirt covered Tiles!");
    }

    protected virtual void Update()
    {
        // Depending on the state of the card, determine behaivor
        switch (myState)
        {
            case CardState.Inactive:
                // Ensure all things are ready for being played
                selectedTile = null;
                selectedUnit = UnitSelector.Instance.selectedUnit;
                unitsHighligthed = false;
                tilesHighligthed = false;
                cardExecutionCalled = false;

                break;
            case CardState.VerifyUnitSelection:
                // Verify if the selected unit is legal
                VerifyUnitSelection(CardTargetFinder.FindLegalUnits(this));

                break;
            case CardState.SelectingUnit:
                // Select a new unit
                // This state is only entered after a failed verification
                if (!unitsHighligthed)
                {
                    List<Tile> tilesWithHighlightContent = new();
                    List<Unit> unitsToHighlight = CardTargetFinder.FindLegalUnits(this);
                    foreach (Unit u in unitsToHighlight)
                    {
                        tilesWithHighlightContent.Add(u.standingOn);
                    }

                    CardTargetFinder.HighlightContent(tilesWithHighlightContent, true, false);
                    unitsHighligthed = true;
                }
                SelectUnit();

                break;
            case CardState.SelectingTile:
                // Select a tile to attempt to use the card on
                if (!tilesHighligthed)
                {
                    CardTargetFinder.UnhighlightAllContent();
                    selectedUnit.standingOn.Highlight(Color.blue);
                    HighlightTiles();
                    tilesHighligthed = true;
                }
                SelectTile();

                break;
            case CardState.VerifyTileSelection:
                // Verify if the selected tile is legal
                VerifyTileSelection(CardTargetFinder.FindLegalTiles(this));

                break;
            case CardState.Executing:
                // Execute the cards behaivour
                if (!cardExecutionCalled)
                {
                    ExecuteBehaivour(selectedTile, selectedUnit);
                    DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.Executing, "Playing card!");
                    cardExecutionCalled = true;
                }
                break;
            case CardState.Finished:
                // Let the card manager know the card is finished and can go to discard
                CardManager.Instance.CardEffectComplete();
                MovementManager.Instance.takingMoveAction = true;
                myState = CardState.Inactive;
                GameStats.Instance.IncreaseCardsPlayed();
                CardTargetFinder.UnhighlightAllContent();
                if (UnitSelector.Instance.selectedUnit != null)
                {
                    UnitSelector.Instance.UpdateSelectedUnit(UnitSelector.Instance.selectedUnit, false, true);
                    UnitSelector.Instance.HighlightAllTilesMovableTo(true);
                }
                ActiveCard.Instance.cardBeingPlayed = null;
                tilesHighligthed = false;
                unitsHighligthed = false;
                cardExecutionCalled = false;
                DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.Inactive, "--");

                break;
            default:
                Debug.LogError("Reached end of state-machine, new case not added to switch?");
                Debug.Log("Going inactive");
                myState = CardState.Inactive;

                break;
        }
    }

    protected virtual void VerifyUnitSelection(List<Unit> legalUnits)
    {
        // If no unit is selected - go to select unit.
        if (selectedUnit == null)
        {
            HandleIllegalSelection("No robot selected, please select a unit",
                                    "Select a Robot.");

            return;
        }

        if (!legalUnits.Contains(selectedUnit))
        {
            HandleIllegalSelection("Illegal unit selected", "Please select a highlighted Robot");

            return;
        }

        // If reached this bit of the code, the bot is verified and get's to cast the card.
        myState = CardState.SelectingTile;
        DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.SelectingTile, "Select a Tile");

        void HandleIllegalSelection(string errorMessage, string cardStateText)
        {
            //Debug.Log(errorMessage);
            selectedUnit = null;
            myState = CardState.SelectingUnit;
            DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.VerifyUnitSelection, cardStateText);
            GridManager.Instance.UnhighlightAll();
            unitsHighligthed = false;
        }
    }

    protected virtual void SelectUnit()
    {
        // Click on a tile
        // If it has an occupant
        // That is now the selected unit
        // Send to verify
        if (Input.GetMouseButtonDown((int)MouseButton.Left))
        {
            Vector3 mousePosition = Input.mousePosition;
            Ray ray = Camera.main.ScreenPointToRay(mousePosition);
            RaycastHit hit;

            if (Physics.Raycast(ray, out hit))
            {
                if (hit.collider != null)
                {
                    Tile clickedTile = null;

                    if (hit.collider.gameObject.TryGetComponent<Unit>(out Unit clickedUnit))
                    {
                        clickedTile = clickedUnit.standingOn;
                        PickUnit(clickedTile);
                        return;
                    }

                    if (hit.collider.gameObject.TryGetComponent<Tile>(out clickedTile))
                    {
                        if (clickedTile.occupant != null)
                        {
                            PickUnit(clickedTile);
                            return;
                        }
                    }
                }
            }

            // Also check UI elements for the mini potraits
            PointerEventData ped = new PointerEventData(GraphicsRayCastAssistance.Instance.eventSystem);
            ped.position = Input.mousePosition;
            List<RaycastResult> results = new List<RaycastResult>();
            GraphicsRayCastAssistance.Instance.caster.Raycast(ped, results);
            foreach (RaycastResult r in results)
            {
                if (r.gameObject.TryGetComponent<UnitMiniPanel>(out UnitMiniPanel ump))
                {
                    PickUnit(ump.myUnit.standingOn);
                    return;
                }
            }

            void PickUnit(Tile onTile)
            {
                selectedUnit = onTile.occupant;
                UnitSelector.Instance.UpdateSelectedUnit(selectedUnit);
                myState = CardState.VerifyUnitSelection;
            }
        }
    }

    protected virtual void HighlightTiles()
    {
        // Find legal tiles
        List<Tile> legalTiles = CardTargetFinder.FindLegalTiles(this);

        // Deterime what to highlight
        bool highlightDirt = hasToTargetDirtTiles;
        bool highlightUnits = hasToTargetOccupiedTiles;

        // Highlight relevant data
        CardTargetFinder.HighlightContent(legalTiles, highlightUnits, highlightDirt);
    }

    protected virtual void SelectTile()
    {
        // Click on a tile, unit, or dirt
        // That is now the selected tile
        // Send to verify
        if (Input.GetMouseButtonDown((int)MouseButton.Left) && !CardManager.Instance.isDisplaying)
        {
            Vector3 mousePosition = Input.mousePosition;
            Ray ray = Camera.main.ScreenPointToRay(mousePosition);
            RaycastHit hit;

            if (Physics.Raycast(ray, out hit))
            {
                if (hit.collider != null)
                {
                    Tile clickedTile = null;

                    if (hit.collider.gameObject.TryGetComponent<Dirt>(out Dirt clickedDirt))
                    {
                        clickedTile = clickedDirt.myTile;
                        PickTile(clickedTile);
                        return;
                    }

                    if (hit.collider.gameObject.TryGetComponent<Unit>(out Unit clickedUnit))
                    {
                        clickedTile = clickedUnit.standingOn;
                        PickTile(clickedTile);
                        return;
                    }

                    if (hit.collider.gameObject.TryGetComponent<Tile>(out clickedTile))
                    {
                        PickTile(clickedTile);
                        return;
                    }
                }
            }

            // Also check UI elements for the mini potraits
            PointerEventData ped = new PointerEventData(GraphicsRayCastAssistance.Instance.eventSystem);
            ped.position = Input.mousePosition;
            List<RaycastResult> results = new List<RaycastResult>();
            GraphicsRayCastAssistance.Instance.caster.Raycast(ped, results);
            foreach (RaycastResult result in results)
            {
                if (result.gameObject.TryGetComponent<UnitMiniPanel>(out UnitMiniPanel ump))
                {
                    PickTile(ump.myUnit.standingOn);
                    return;
                }
            }

            void PickTile(Tile thisTile)
            {
                selectedTile = thisTile;
                myState = CardState.VerifyTileSelection;
            }
        }
    }

    protected virtual void VerifyTileSelection(List<Tile> legalTargets)
    {
        // If no selected tile is passed
        if (selectedTile == null)
        {
            HandleIllegalSelection("No tile selected for verification - error in structure",
                                    "Please select a highlighted Tile");
            return;
        }

        // We already gather all legal tiles to highlight them, no need to do that twice
        if (!legalTargets.Contains(selectedTile))
        {
            HandleIllegalSelection("Illegal target selected",
                                    "Please select a highlighted target");
            return;
        }

        // If reached this bit of the code, the card is valid and can be executed
        myState = CardState.Executing;

        // When an illegal selection is made, log it and return to the selection stage
        void HandleIllegalSelection(string errorMessage, string cardStateText)
        {
            //Debug.Log(errorMessage);
            DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.VerifyTileSelection, cardStateText);
            myState = CardState.SelectingTile;
            CardTargetFinder.UnhighlightAllContent();
            tilesHighligthed = false;
        }
    }

    // Called to start playing the process of playing the card
    public virtual void Play()
    {
        // Since units may be selected before playing, start by verifying the selected unit to play the card with.
        myState = CardState.VerifyUnitSelection;
        DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.SelectingUnit, "Select a unit");

        // Disable the ability to move units while playing cards
        MovementManager.Instance.takingMoveAction = false;
    }


    public abstract void ExecuteBehaivour(Tile onTile, Unit byUnit);

    /// <summary>
    /// Left Abstract to force reminder when implementing new subclass
    /// Call this once the card behaivour is complete
    /// </summary>
    public abstract void ConfirmCardExecuted();

    public void CancelPlay()
    {
        DEBUGCardStateUI.Instance.DEBUGUpdateUI(CardState.Inactive, "--");

        selectedTile = null;
        selectedUnit = null;
        myState = CardState.Inactive;

        ActiveCard.Instance.cardBeingPlayed = null;
        MovementManager.Instance.takingMoveAction = true;

        CardTargetFinder.UnhighlightAllContent();
        UnitSelector.Instance.UpdateSelectedUnit(UnitSelector.Instance.selectedUnit);
    }
}]
```
</details>

<details>
<summary>CardTargetFinder.cs</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using Unity.VisualScripting;
using UnityEngine;

public static class CardTargetFinder
{
    public static List<Unit> FindLegalUnits(Card forCard)
    {
        // Find legal units
        List<Unit> legalUnits = new();
        if (forCard.requiredSpecialization != BotSpecialization.None)
            legalUnits = UnitStorage.Instance.playerUnits.Where(u => u.mySpecialization == forCard.requiredSpecialization).ToList();
        else
            legalUnits = UnitStorage.Instance.playerUnits;

        #region No legal targets behaivour
        // If no legal units, put card back in hand
        /*if (legalUnits.Count == 0)
        {
            //Debug.Log("No legal unit found, removing card from played");
            CancelPlay();
            CardManager.instance.ClearActiveCard();
            return;
        }*/
        #endregion

        // Remove duplicates
        legalUnits = legalUnits.Distinct().ToList();

        return legalUnits;
    }

    public static List<Tile> FindLegalTiles(Card forCard)
    {
        // Find legal tiles
        List<Tile> legalTiles = new();

        legalTiles.AddRange(GridManager.Instance.tiles);
        List<Tile> tilesToRemove = new();
        // Remove illegal tiles
        foreach (Tile t in legalTiles)
        {
            // Wall check
            if(forCard.canNotTargetWalls && t.walled)
                tilesToRemove.Add(t);

            // Dirt Check
            if (forCard.hasToTargetDirtTiles && !t.containsDirt)
                tilesToRemove.Add(t);

            if (forCard.canNotTargetDirtTiles && t.containsDirt)
                tilesToRemove.Add(t);

            // Unit Check
            if (forCard.hasToTargetOccupiedTiles && !t.occupied)
                tilesToRemove.Add(t);

            if (forCard.canNotTargetOccupiedTiles && t.occupied)
                tilesToRemove.Add(t);

            // Range check
            if (Pathfinding.GetDistance(t, forCard.selectedUnit.standingOn) > forCard.range || Pathfinding.GetDistance(t, forCard.selectedUnit.standingOn) == 0)
                tilesToRemove.Add(t);
        }

        foreach (Tile t in tilesToRemove)
        {
            if (legalTiles.Contains(t))
                legalTiles.Remove(t);
        }

        #region No legal targets behaivour
        // If no legal tiles, put card back in hand
        /*if (legalTiles.Count == 0)
        {
            //Debug.Log("No legal tile found, removing card from played");
            CancelPlay();
            CardManager.instance.ClearActiveCard();
            return;
        }*/
        #endregion

        // Remove duplicates
        legalTiles = legalTiles.Distinct().ToList();

        return legalTiles;
    }

    public static void HighlightContent(List<Tile> onTiles, bool highlightUnits, bool highlightDirt)
    {
        foreach (Tile t in onTiles)
        {
            t.Highlight();

            if (highlightUnits)
            {
                if(t.occupant != null)
                {
                    t.occupant.Highlight();
                    List<UnitMiniPanel> panelsToHighlight = UnitStorage.Instance.playerPanels.Where(ump => ump.myUnit == t.occupant).ToList();
                    if (panelsToHighlight.Count != 0)
                        panelsToHighlight[0].Highlight();
                }
            }

            if(highlightDirt)
            {
                if (t.dirt != null)
                    t.dirt.Highlight();
            }
        }
    }

    public static void UnhighlightAllContent()
    {
        GridManager.Instance.UnhighlightAll();
        foreach (Unit u in UnitStorage.Instance.playerUnits)
            u.UnHighlight();
        foreach (Unit u in UnitStorage.Instance.enemyUnits)
            u.UnHighlight();
        foreach (UnitMiniPanel ump in UnitStorage.Instance.playerPanels)
            ump.UnHighlight();
    }
}

```

</details>

## I also worked with
  
<b>AI Behaivour </b>

During the AIs turn it cycles through all enemy units and locates the closest player unit, moves it closer towards that player, and attacks if in range.


<b>Audio Manager </b>

Objectpooling AudioSoruces to frontload the resource allocation. Updates each source when audio settings are updated.


<b>Camera Manager </b>

The player can move the camera using either WASD, or by holding down the mouse wheel and draggin the mouse.

The camera will automatically move to each "targeted" unit unless the player has specifically moved the camera.

