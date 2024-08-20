# About

 <img src="images\mash-off-monsterrumble_logo.png" alt="Logo for Mash-Off: Monster Rumble" width=75%>

MASH-OFF: Monster Rumble is a local multiplayer twinstick party game about knocking your opponents out of an arena using various cartoony weapons while they try to do the same to you.

Repository: [[Link]](https://github.com/LadyRonja/ArenaEject)

Itch: [[Link]](https://yrgo-game-creator.itch.io/mash-off-monster-rumble)

# My Responsibilities

## <b>Camera</b> 
I worked on a camera that smoothly keeps all players in frame at all times.

The script tracks how far away each player is from each other and calculates how far away the camera needs to be to keep them in frame given it's current FOV and angle.

It moves smoothly and speeds up automatically so players never leave frame. It also keeps max distances caches for a short amount of time to feel less frantic.

Camera shake is also smoothed to help players still be able to track what's going on while adding game juice.

<i>Click the dropdown arrow below to see the code</i>
<details>
<summary>CameraControllser.cs</summary>
  
```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using UnityEditor.Rendering;
using UnityEngine;

public enum CameraModes { STILL, TRACK_OBJECTS, SHAKE, RESETTING}

public struct TrackPoint{
    public TrackPoint(Vector3 position, DateTime savedAt, float lifeTime)
    {
        this.position = position;
        this.savedAt= savedAt;
        this.lifeTime = lifeTime;
    }

    public Vector3 position;
    public DateTime savedAt;
    private float lifeTime;
    public bool Expired { get => DateTime.Now > savedAt.AddSeconds(lifeTime); }
}

public class CameraController : MonoBehaviour
{
    private static CameraController instance;
    public static CameraController Instance { get => instance; }

    [Header("Generic")]
    [SerializeField] CameraModes cameraMode = CameraModes.STILL;
    [Space]
    [SerializeField] float xAngleDeg = 35f;
    [Space]
    [SerializeField] bool displayDebugs = false;
    Vector3 initialPosition = Vector3.zero;
    Quaternion initialRotation = Quaternion.identity;



    Vector3 targetPos = Vector3.zero;

    [Header("Track Objects")]
    [Range(0.01f, 0.99f)][SerializeField] float trackSpeedMinimum = 0.1f;
    [SerializeField] float maxZoom = 3f;
    [Space]
    [SerializeField] float paddingWidth = 2f;
    [SerializeField] float paddingTop = 1f;
    [SerializeField] float paddingBottom = 8f;
    [Space]
    [SerializeField] float lingerTime = 0.5f;
    [Space]
    List<Transform> transformsToTrack;
    private List<TrackPoint> delayedExtremes= new List<TrackPoint>();
    float camTrackSpeed = 0.1f; // dynamically changes, initialized value only sets start speed, hence not serialized

    [Header("Shake Debug")]
    public float DEBUG_ShakeAmount = 0.3f;
    public float DEBUG_ShakeDuration = 0.1f;


    private void Start()
    {
        if(instance == null || instance == this)
        {
            instance = this;
            targetPos = transform.position;
            transform.rotation = Quaternion.Euler(xAngleDeg, 0f, 0f);
            transformsToTrack = new();

            initialPosition = transform.position;
            initialRotation = transform.rotation;
        }
        else
        {
            Destroy(this.gameObject);
        }
    }

    private void FixedUpdate()
    {
        switch (cameraMode)
        {
            case CameraModes.TRACK_OBJECTS:
                TrackObjects();
                break;
            case CameraModes.RESETTING:
            case CameraModes.STILL:
            case CameraModes.SHAKE:
                break;
            default:
                break;
        }
    }

    public void AddShake(float amount = 0.1f, float duration = 0.3f)
    {
        StartCoroutine(ShakeRoutine(amount, duration));
    }

    private IEnumerator ShakeRoutine(float amount, float duration)
    {
        float timePassed = 0f;

        // Generate slightly offset points to move to while shaking
        int shakeTimes = 6;
        List<int> xOffSet = new();
        List<int> zOffSet = new();
        int lastX = 0;
        int lastZ = 0;

        // Ensure the camera doesn't linger in one corner
        for (int i = 0; i < shakeTimes; i++)
        {
            int newX = ReturnRandPosOrNeg();
            int newZ = ReturnRandPosOrNeg();

            while (newX == lastX && newZ == lastZ)
            {
                newX = ReturnRandPosOrNeg(); ;
                newZ = ReturnRandPosOrNeg();
            }
            xOffSet.Add(newX);
            zOffSet.Add(newZ);
            lastX = newX;
            lastZ = newZ;
        }

        // Smoothly move the camera to each location in order
        for (int i = 0; i < shakeTimes; i++)
        {
            timePassed = 0;
            Vector3 startPos = transform.position;
            Vector3 endPos = new Vector3(
                                            targetPos.x + amount * xOffSet[i], 
                                            targetPos.y, 
                                            targetPos.z + amount * zOffSet[i] 
                                            );
           
            while (timePassed < duration/shakeTimes)
            {
                transform.position = Vector3.Lerp(startPos, endPos, (timePassed / (duration/shakeTimes)));

                timePassed += Time.deltaTime;
                yield return null;
            }
        }

        // Reset the camera to the latest cameraMode (always tracking here)
        cameraMode = CameraModes.TRACK_OBJECTS;

        // Returns either 1, or -1
        int ReturnRandPosOrNeg()
        {
            int output = (UnityEngine.Random.Range(0, 2) == 1) ? 1 : -1;
            return output;
        }
    }

    public void StartTrackingObjects(List<Transform> objectsToTrack)
    {
        transformsToTrack = objectsToTrack;
        delayedExtremes.Clear();
        cameraMode = CameraModes.TRACK_OBJECTS;
    }

    private void TrackObjects()
    {
        EnsureLegalTrackedObjects();
        if (transformsToTrack.Count < 1) {
            StartCoroutine(ResetCamera());
            return; 
        }

        // Find the largest and smallest extremes of each tracked object
        Vector3 targetMaxs = transformsToTrack[0].position;
        Vector3 targetMins = transformsToTrack[0].position;
        for (int i = 0; i < transformsToTrack.Count; i++) {
            _ScanAndSetExtremes(transformsToTrack[i].position);
        }

        // Save max positions for a linger time for a less frantic camera
        delayedExtremes.Add(new TrackPoint(targetMaxs, DateTime.Now, lingerTime));
        delayedExtremes.Add(new TrackPoint(targetMins, DateTime.Now, lingerTime));       
        delayedExtremes.RemoveAll(de => de.Expired);
        foreach (TrackPoint tp in delayedExtremes) {
            _ScanAndSetExtremes(tp.position);
        }

        // Find middle point between players
        Vector3 centerPoint = (targetMaxs + targetMins) * 0.5f;
        targetPos = centerPoint;

        // Padding
        float paddedRightMost = targetMaxs.x + paddingWidth;
        float paddedLeftMost = targetMins.x - paddingWidth;
        float paddedTopMost = targetMaxs.z + paddingTop;
        float paddedBottomMost = targetMins.z - paddingBottom;

        // Frustrums
        float frustrumDesiredWidth = paddedRightMost - paddedLeftMost;
        float frustrumDesiredHeight = paddedTopMost - paddedBottomMost;
        frustrumDesiredWidth = Mathf.Max(maxZoom, frustrumDesiredWidth);
        frustrumDesiredHeight = Mathf.Max(maxZoom, frustrumDesiredHeight);

        // Determine the zoom distance needed to encompass the largest frustrum
        float distanceHypotonuseHeight = (frustrumDesiredHeight * 0.5f) / (Mathf.Tan(Camera.main.fieldOfView * 0.5f * Mathf.Deg2Rad));
        float distanceHypotonuseWidth = (frustrumDesiredWidth * 0.5f) / (Mathf.Tan(Camera.main.fieldOfView * 0.5f * Mathf.Deg2Rad));
        float distanceHypotonuse = Mathf.Max(distanceHypotonuseHeight, distanceHypotonuseWidth);

        targetPos -= transform.forward * distanceHypotonuse;

        #region Debug Drawing
        if (displayDebugs) {
            // Draw square using Debug.DrawLine(s) to display the currently tracked frustrum 
            float squareY = centerPoint.y;

            Vector3 squareOriginalCornerNW = new Vector3(targetMins.x, squareY, targetMaxs.z);
            Vector3 squareOriginalCornerNE = new Vector3(targetMaxs.x, squareY, targetMaxs.z);
            Vector3 squareOriginalCornerSE = new Vector3(targetMaxs.x, squareY, targetMins.z);
            Vector3 squareOriginalCornerSW = new Vector3(targetMins.x, squareY, targetMins.z);
            DrawDebugSquare(squareOriginalCornerNW, squareOriginalCornerNE, squareOriginalCornerSE, squareOriginalCornerSW, Color.magenta);
        }
        #endregion

        // If any of the extremes are outside of the cameras view,
        // increase the camera interporlationspeed
        // Otherwise decrease it
        // Then clamp it
        float camAcceleration = 0.05f; // Magic number, fast enough to catch most items, slow enough to not be sowewhat smooth
        if (!_IsInViewRange(targetMins) || !_IsInViewRange(targetMaxs)) {
            camTrackSpeed += camAcceleration;
        }
        else {
            camTrackSpeed -= camAcceleration;
        }

        // Clamp track speed interporlation (values outside of 0-1 does nothing)
        camTrackSpeed = Mathf.Clamp(camTrackSpeed, trackSpeedMinimum, 1f);

        // Move camera
        transform.position = Vector3.Lerp(transform.position, targetPos, camTrackSpeed);

        // Determine if a point in the world is within the screens viewport.
        bool _IsInViewRange(Vector3 point)
        {
            Vector3 viewportPoint = Camera.main.WorldToViewportPoint(point);

            if (viewportPoint.x > 1 || viewportPoint.x < 0)
            {
                return false;
            }
            else if (viewportPoint.y > 1 || viewportPoint.y < 0)
            {
                return false;
            }
            else if(viewportPoint.z < 0)
            {
                return false;
            }

            return true;
        }

        // Scan for extreme mins and maxs,
        // OBS: Mutates scoped state directly
        void _ScanAndSetExtremes(Vector3 scannedPosition)
        {
            targetMaxs.x = Mathf.Max(targetMaxs.x, scannedPosition.x);
            targetMaxs.y = Mathf.Max(targetMaxs.y, scannedPosition.y);
            targetMaxs.z = Mathf.Max(targetMaxs.z, scannedPosition.z);

            targetMins.x = Mathf.Min(targetMins.x, scannedPosition.x);
            targetMins.y = Mathf.Min(targetMins.y, scannedPosition.y);
            targetMins.z = Mathf.Min(targetMins.z, scannedPosition.z);
        }
    }

    private void EnsureLegalTrackedObjects()
    {
        List<Transform> missingTrackedObjects = new();
        foreach (Transform trackedObject in transformsToTrack)
        {
            if (trackedObject == null)
            {
                missingTrackedObjects.Add(trackedObject);
            }
        }
        foreach (Transform trackedObject in missingTrackedObjects)
        {
            transformsToTrack.Remove(trackedObject);
        }
    }

    private IEnumerator ResetCamera(float inSeconds = 1.5f)
    {
        cameraMode = CameraModes.RESETTING;

        float timePassed = 0;
        Vector3 startPos = transform.position;
        Vector3 endPos = initialPosition;
        Quaternion startRot = transform.rotation;
        Quaternion endRot = initialRotation;

        while (timePassed < inSeconds)
        {
            transform.position = Vector3.Lerp(startPos, endPos, (timePassed / inSeconds));
            transform.rotation = Quaternion.Lerp(startRot, endRot, (timePassed / inSeconds));

            timePassed += Time.deltaTime;
            yield return null;
        }
        transform.position = endPos;
        transform.rotation = endRot;

        cameraMode = CameraModes.STILL;
        yield return null;
    }

    private void DrawDebugSquare(Vector3 squareCornerNW, Vector3 squareCornerNE, Vector3 squareCornerSE, Vector3 squareCornerSW, Color color)
    {
        float squareDrawTime = Time.deltaTime;
        Debug.DrawLine(squareCornerNW, squareCornerNE, color, squareDrawTime);
        Debug.DrawLine(squareCornerNE, squareCornerSE, color, squareDrawTime);
        Debug.DrawLine(squareCornerSE, squareCornerSW, color, squareDrawTime);
        Debug.DrawLine(squareCornerSW, squareCornerNW, color, squareDrawTime);
    }
}
```
</details>

## Audio Manager 
I repurposed a AudioManger I wrote for another project which creates and updates an object pool of audio sources for use of soundeffects and music.

If more audio sources are needed the pool is expanded, however so far the sources generated on initalization has always been enough.

The volume of the sources updates whenever the volume settings are changed.

<i>Click the dropdown arrow below to see the code</i>
<details>
<summary>AudioHandler.cs</summary>
  
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.SceneManagement;

public class AudioHandler : MonoBehaviour
{
    private static AudioHandler instance;
    public static AudioHandler Instance { get => GetInstance(); private set => instance = value; }
    public static bool deleteOtherSources = true;

    [Header("Music")]
    [SerializeField] AudioClip music;
    AudioSource musicPlayer;

    [Header("Effects")]
    [SerializeField] int initialEffectSourceCount = 10;
    List<AudioSource> effectSources = new();

    private void Awake()
    {
        #region Singleton
        if (instance == null || instance == this)
            instance = this;
        else
        {
            Destroy(this.gameObject);
            return;
        }
        #endregion

        if (deleteOtherSources)
            DestroyAllOtherSources();

        SetUpMusicPlayer();
        ExpandSorceCount(initialEffectSourceCount);


        if (gameObject != null)
            SceneManager.sceneLoaded += delegate { PickMusic(); };
    
        DontDestroyOnLoad(gameObject);
    }

    public static void PlaySoundEffect(AudioClip clipToPlay)
    {
        if (clipToPlay == null) return;
        if (!Application.isPlaying) return;

        // Loop through all effect sources in Instance until a free one plays
        // If no free source is available, make more sources

        // Potential optimization would be to just play from one of the new sources instantly, rather than looping through again.
        // The reason this isn't done, is because Unity has previously created/destroyed objects at end of frame while continuing the stack

        bool soundStartedPlaying = false;
        while (!soundStartedPlaying)
        {
            foreach (AudioSource s in Instance.effectSources)
            {
                if (s.isPlaying)
                    continue;

                s.volume = Settings.EFFECT_VOULME;

                s.PlayOneShot(clipToPlay);
                soundStartedPlaying = true;
                break;
            }

            if (!soundStartedPlaying)
                Instance.ExpandSorceCount(10);
        }

    }

    public static void PlayRandomEffectFromList(List<AudioClip> possibleClips)
    {
        if (possibleClips == null) return;
        if (possibleClips.Count == 0) return;

        int rand = Random.Range(0, possibleClips.Count);
        AudioClip clipToPlay = possibleClips[rand];

        PlaySoundEffect(clipToPlay);
    }

    private void SetUpMusicPlayer()
    {
        PickMusic();

        if (musicPlayer == null)
            musicPlayer = this.gameObject.AddComponent<AudioSource>();

        musicPlayer.loop = true;
        musicPlayer.volume = Settings.MUSIC_VOULME;

        if (!musicPlayer.isPlaying)
        {
            musicPlayer.clip = music;
            musicPlayer.Play();
        }
    }

    private void PickMusic()
    {
        AudioClip prevClip = null;
        if(musicPlayer!= null) { prevClip = musicPlayer.clip; }

        if (SceneManager.GetActiveScene().name == Paths.START_SCENE_NAME)
            music = Resources.Load<AudioClip>(Paths.START_MENU_MUSIC);
        else
            music = Resources.Load<AudioClip>(Paths.DEFAULT_GAMEPLAY_MUSIC);

        if(music != prevClip)
        {
            if (musicPlayer == null) return;

            musicPlayer.Stop();
            musicPlayer.clip = music;
            musicPlayer.loop = true;
            musicPlayer.Play();
        }
    }

    private void ExpandSorceCount(int amount)
    {
        for (int i = 0; i < amount; i++)
        {
            GameObject newSourceObject = new GameObject("Effect Source");
            newSourceObject.transform.parent = transform;
            AudioSource newSource = newSourceObject.AddComponent<AudioSource>();

            newSource.volume = Settings.EFFECT_VOULME;

            effectSources.Add(newSource);
        }
    }

    private void DestroyAllOtherSources()
    {
        AudioSource[] oldSources = Object.FindObjectsOfType(typeof(AudioSource)) as AudioSource[];
        foreach (AudioSource o in oldSources)
        {
            AudioHandler thisCheck = o.GetComponent<AudioHandler>();
            if (thisCheck != null)
                if (thisCheck == this)
                    return;

            Destroy(o.transform.gameObject);
        }
    }

    public void UpdateMusicVolume(float volume)
    {
        volume = Mathf.Clamp(volume, 0f, 1f);

        musicPlayer.volume = volume;
    }

    public void UpdateEffectVolume(float volume)
    {
        volume = Mathf.Clamp(volume, 0f, 1f);
        foreach (AudioSource s in effectSources)
        {
            s.volume = volume;
        }
    }

    public void TuneOutMusic()
    {
        StartCoroutine(TurnOffMusic());
    }

    private IEnumerator TurnOffMusic()
    {
        float startVol = musicPlayer.volume;
        float timePassed = 0;
        float timeToQuiet = 1f;

        while (timeToQuiet > timePassed)
        {
            musicPlayer.volume = Mathf.Lerp(startVol, 0, (timePassed / timeToQuiet));
            timePassed += Time.deltaTime;

            yield return null;
        }
        musicPlayer.volume = 0;
        yield return null;
    }


    private static AudioHandler GetInstance()
    {
        if (instance != null)
            return instance;

        if (!Application.isPlaying)
            return null;

        GameObject newManager = new GameObject("AudioManager");
        instance = newManager.AddComponent<AudioHandler>();
        return instance;
    }
}
```
</details>

## <b>Character Animator</b>
I wrote the character animator which is inspired by [this tutorial](https://www.youtube.com/watch?v=ZwLekxsSY3Y) by youtuber Tarodev

<i>Click the dropdown arrow below to see the code</i>
<details>
<summary>PlayerAnimationManager.cs</summary>
  
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;


public class PlayerAnimationManager : MonoBehaviour
{
    [HideInInspector] public Animator animator;
    private Movement _movementController;
    private WeaponUser _weaponUser;
    private GroundChecker _groundCheck;
    private KnockBackHandler _knockBackHandler;
    private int currentState = 0;
    private bool animationStateIsLocked = false;

    private Dictionary<int, int> animationsWithWeaponPairings = new();

    private int defaultAnimation = 0;
    private static string defaultAnimationName = "rig|anim_player_idle";
    private static readonly int IDLE = Animator.StringToHash("rig|anim_player_idle");
    private static readonly int IDLE_GUN = Animator.StringToHash("rig|anim_idle_weapon");
    private static readonly int RUN = Animator.StringToHash("rig|anim_player_run");
    private static readonly int RUN_GUN = Animator.StringToHash("rig|anim_player_run_with_gun");
    private static readonly int JUMP = Animator.StringToHash("rig|anim_jump_start");
    private static readonly int JUMP_GUN = Animator.StringToHash("rig|anim_jump_weapon_start");
    private static readonly int FALL = Animator.StringToHash("rig|anim_jump_midair");
    private static readonly int FALL_GUN = Animator.StringToHash("rig|anim_jump_weapon_midair");
    private static readonly int KNOCKEDBACK = Animator.StringToHash("rig|anim_damage");

    // Unimplemented
    private static readonly int LAND = Animator.StringToHash("rig|anim_jump_end");
    private static readonly int LAND_GUN = Animator.StringToHash("rig|anim_jump_end");



    private void Start()
    {
        animator = GetComponentInChildren<Animator>();
        if (animator == null)
        {
            Debug.LogError("AnimationHandler unable To find player animator component!");
        }

        if (!TryGetComponent<Movement>(out _movementController))
        {
            Debug.LogError("AnimationHandler unable To find player Movement script!");
        }

        if (!TryGetComponent<WeaponUser>(out _weaponUser))
        {
            Debug.LogError("AnimationHandler unable To find player WeaponUser script!");
        }

        if (!TryGetComponent<GroundChecker>(out _groundCheck))
        {
            Debug.LogError("AnimationHandler unable To find player GroundCheck script!");
        }

        if (!TryGetComponent<KnockBackHandler>(out _knockBackHandler))
        {
            Debug.LogError("AnimationHandler unable To find player KnockBackHandler script!");
        }
        else
        {
            _knockBackHandler.OnKnockbackRecieved += PlayKnockbackAnimation;
            _knockBackHandler.OnKnockbackComplete += UnlockAnimationState;
        }

        defaultAnimation = IDLE;
        DecalareAnimationPairings();

    }

    private void FixedUpdate()
    {
        ManageAnimations();
    }

    private void OnDestroy()
    {
        _knockBackHandler.OnKnockbackRecieved -= PlayKnockbackAnimation;
        _knockBackHandler.OnKnockbackComplete -= UnlockAnimationState;
    }

    private void DecalareAnimationPairings()
    {
        animationsWithWeaponPairings = new();

        animationsWithWeaponPairings.Add(IDLE, IDLE_GUN);
        animationsWithWeaponPairings.Add(RUN, RUN_GUN);
        animationsWithWeaponPairings.Add(JUMP, JUMP_GUN);
        animationsWithWeaponPairings.Add(FALL, FALL_GUN);
    }

    private void ManageAnimations()
    {
        if (!ComponentsVerified()) { return; }

        int desiredAnimationState = GetDesiredAnimationState();
        TrySetAnimation(desiredAnimationState);
    }

    private int GetDesiredAnimationState()
    {
        if (!ComponentsVerified()) { return defaultAnimation; }
        if (animationStateIsLocked) { return currentState; }

        int animationToReturn = GetBaseAnimation();

        // Check if this animation has aweapon variant
        if(_weaponUser.carriedWeapon != null)
        {
            foreach (var key in animationsWithWeaponPairings.Keys)
            {
                if (key == animationToReturn)
                {
                    return animationsWithWeaponPairings[key];
                }
            }
        }

        return animationToReturn;

        // Get base animation, ignoring weather or not a weapon is carried
        int GetBaseAnimation()
        {
            if (!ComponentsVerified()) { return defaultAnimation; }

            int animationToReturn = defaultAnimation;

            // Knockback moved to be ttriggered only by event
            /*
            // Knocked back
            if (_knockBackHandler.KnockedBack)
            {
                return KNOCKEDBACK;
            }*/

            // Airborn
            if (!_groundCheck.IsGrounded)
            {
                if (_movementController.RB.velocity.y > 0)
                {
                    return JUMP;
                }
                else
                {
                    return FALL;
                }
            }

            // Movement
            if (_movementController.RB.velocity.sqrMagnitude <= 0.1f)
            {
                return IDLE;
            }
            else
            {
                return RUN;
            }
        }
    }

    private bool ComponentsVerified()
    {
        if (animator == null) { return false; }
        if (_movementController == null) { return false; }
        if (_weaponUser == null) { return false; }
        if (_groundCheck == null) { return false; }
        if (_knockBackHandler == null) { return false; }

        return true;
    }

    private void TrySetAnimation(int animation)
    {
        if (animationStateIsLocked) { return; }
        if (currentState == animation) {return; }

        animator.CrossFade(animation, 0f, 0);
        currentState = animation;
    }

    private void PlayKnockbackAnimation()
    {
        TrySetAnimation(KNOCKEDBACK);
        LockAnimationState();
    }

    private void LockAnimationState()
    {
        animationStateIsLocked = true;
    }

    private void UnlockAnimationState()
    {
        animationStateIsLocked = false;
    }
}
```
</details>


## I also worked with

<b>Character controls</b>

Various aspects of the playermovent, including settign up the new Unity input system

<b>The Lobby and Game Start</b>

Having players be able to join/leave a game lobby and assigning them a controller index.

Loading each match properly, including the right player models for the right players.

<b>The Weapon and Pickup system</b>

The framework for how the player ineracts with weapons and pickups.

<b>Main menu UI functionality</b>

Setting up and connecting the menu's in Unity to allow propper controller support.



## [Back to Main Page](https://github.com/LadyRonja/portfolio/blob/main/README.md)
