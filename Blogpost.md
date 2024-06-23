# Blog Post: Implementing Bezier Curve Mechanics to a Match-3 Blast Game
## Introduction
For my term project, I aim to develope a match-3 game and implement Bezier curves for a special item called copter or propeller. This project needs a comprehensive development process, including the design and implementation of game mechanics, user interface, level generation, and additional features. Throughout this blog post, I will discuss the difficulties faced during the project, the solutions employed to overcome them, design choices made and features aimed to implement. Additionally, I will share intermediate visuals and a detailed project demonstration videos. All visuals are taken from Peak games' Toon Blast game, and they allow me to use their assets for my term project.

## Initial Goals and Features
The initial objectives of my term project:

Basic Match-3 Gameplay Mechanics: Implementing the fundamental match-3 mechanics where players tap tiles to blast a group of tiles.

Boosters: There are many boosters in a match-3 game such as bomb, disco and rockets. In my term project, I will implement propeller as a booster.

Special Tiles (Obstacles): I implemented duck and balloon as an obstacle. These items behave differently from normal cubes.

Sound Effects: Adding sound effects for game actions to enhance the overall gaming experience.

Level Goal Mechanic: There are adjustable level goals so that users can play the game by targeting the level goals. If a user could not complete a level within a given amount of move count, level failed.

Propeller/Copter Implementation: I implemented propeller mechanic with the help of bezier curves. Propeller has a start point and an end point. Between these two point, propeller fly smoothly. Propeller also has a shadow to have a better feel of height. Propeller also have a mechanism to change its target smoothly (creating a new bezier curve by using direction vector).

I implemented each item above in my project. I added a screenshot below to show how a level looks:

![1 level-look](https://github.com/ensargoktekin/CENG469-INFO/assets/93570150/c8eaa2b7-4eca-411d-bc91-5ef28dac12c1)

## Challenges Faced and Solutions
Throughout the development process, several challenges arose. Here’s a detailed account of these difficulties and the solutions implemented:

1. Fall Gameplay Mechanics:

Challenge: Implementing an accurate fall mechanic by keeping track of all interactions such as consecutive falls or blastings.

Solution: There was a layer called slot between items and grid. Grid checks availability by asking some information to slot. Then, grid does necessary calculation. Finally, grid notifies slots to fall items.

Calculations in grid:
```c#
/// <summary>
/// Fill corresponding slot with item.
/// </summary>
/// <param name="slot">slot to fill with item</param>
/// <param name="posX">x coordinate of slot</param>
/// <param name="posY">y coordinate of slot</param>
private void FillSlot(Slot slot, int posX, int posY)
{
    for (int i = posY + 1; i < _rowNum; i++)
    {
        if (_slots[i, posX].ItemFalling) // Wait for above slots to fill, then get block from them
        {
            _hasEmptySlot = true;
            return;
        }
        else if (_slots[i, posX].CanProvideItem)
        {
            ItemModel item = _slots[i, posX].ReleaseItem();
            slot.NewItemFall(item);
            return;
        }
        else if (_slots[i, posX].HasItem && !_slots[i, posX].CanProvideItem)
        {
            return;
        }
    }

    // if there is no slot providing an item, create new item
    ProvideItemToSlot(slot);
}
```

Slot Layer (only the actions related with fall mechanism):
```c#
[SerializeField] private ItemModel _item;
[SerializeField] private State _state;

public bool ItemFalling => _state == State.ItemFalling;
public bool HasItem => _state == State.HasItem;
public bool CanProvideItem
{
    get
    {
        if (_state == State.HasItem)
        {
            return _item.CanFall();
        }
        return false;
    }
}

/// <summary>
/// Release item of this slot, but do not destroy it such that released item can be used later.
/// </summary>
/// <returns>released block</returns>
public ItemModel ReleaseItem()
{
    ItemModel releasedItem = _item;
    _item = null;
    _state = State.WaitingItem;
    return releasedItem;
}

/// <summary>
/// Called when a new item is fallen from above to this slot
/// </summary>
/// <param name="item">arrival item</param>
public void NewItemFall(ItemModel item)
{
    _state = State.ItemFalling;
    item.FallTo(transform.position, () =>
    {
        _item = item;
        _state = State.HasItem;
    });
}
```

2. Level Design:

Challenge: Designing levels in an easy way was challenging. We should easily change initial level items to test anything we want.

Solution: I developed a custom level editor tool for unity inspector that allowed for quick adjustments and testing of level configurations. This tool simplified the level design process.

Components to generate a level (later, some calculations are done in Level Generator.cs):

```c#
[SerializeField, Range(3, 9)] public int _width;
[SerializeField, Range(3, 9)] public int _height;
[SerializeField, Range(1, 50)] private int _moveCount;
[SerializeField] private List<GoalPair> _levelGoals;
[SerializeField] private Matrix _grid;
```
How these look in Unity Inspector:

![2 level-generator](https://github.com/ensargoktekin/CENG469-INFO/assets/93570150/8f86046d-ba74-4b99-9484-d321ffe3af0f)

3. Sound Integration:

Challenge: Implement sound effects for multiple objects at any moment of the game. There may be multiple audio clips at the same time.

Solution: I create a singleton Audio Manager to handle all audio actions. Audio manager keeps many audio sources to handle multiple audio clips. Firstly, it gets an available audio sound player. Then, it plays the audio with that sound player.

Here is the implementation:

```c#
public void Play(AudioName audioName)
{
    AudioSource soundPlayer = GetAvailableSoundPlayer();

    soundPlayer.pitch = 1;
    switch (audioName)
    {
        case AudioName.CubePop:
            soundPlayer.PlayOneShot(ReferencesAudio.Inst.CubePop);
            break;
        case AudioName.CubeCollect:
            soundPlayer.PlayOneShot(ReferencesAudio.Inst.CubeCollect);
            break;
        case AudioName.Duck:
            soundPlayer.PlayOneShot(ReferencesAudio.Inst.Duck);
            break;
        case AudioName.Balloon:
            soundPlayer.PlayOneShot(ReferencesAudio.Inst.Balloon);
            break;
    }
}

private AudioSource GetAvailableSoundPlayer()
{
    _currentSoundPlayerIndex++;
    if (_currentSoundPlayerIndex == _maxSoundPlayerIndex)
        _currentSoundPlayerIndex = 0;
    return SoundPlayers[_currentSoundPlayerIndex];
}
```

4. Modularity, Flexibility and Accurate Design Choice:

Challenge: Designing a flexible and reusable code structure for game items.

Solution: The game’s architecture was designed to be flexible, allowing for the easy addition of new features in the future. I used an abstract class as a base item for different game entities. This approach allowed for easy extension and modification of item behaviors, contributing to a more organized and maintainable codebase.

ItemModel abstract class (parent of each item in the game):

```c#
public abstract class ItemModel : MonoBehaviour
{
    [HideInInspector] public ItemType Type;

    /// <summary>
    /// Set parent and initial position
    /// </summary>
    /// <param name="parent">parent to be set</param>
    /// <param name="itemInitParams">init parameters</param>
    public abstract void InitializeItem(Transform parent, Vector3 position, ItemInitParams itemInitParams);

    /// <summary>
    /// Check whether an item can fall
    /// </summary>
    public abstract bool CanFall();

    /// <summary>
    /// Check whether an item can be blasted by tapping
    /// </summary>
    public abstract bool CanBeBlasted();

    /// <summary>
    /// Check whether an item can pop (some objects are never pop and just fall such as duck)
    /// </summary>
    public abstract bool CanPop();

    /// <summary>
    /// Check whether an item is a cube or any other item
    /// </summary>
    public abstract bool IsCube();

    /// <summary>
    /// Called when an item falls
    /// </summary>
    /// <param name="position">target position</param>
    /// <param name="onComplete">callback to invoke after animation</param>
    public abstract void FallTo(Vector3 position, System.Action onComplete);

    /// <summary>
    /// Called when corresponding item pops
    /// </summary>
    public abstract void Damage(DamageData damageData, System.Action itemExplodeCallback);
}
```

## Propeller Mechanic
Propeller mechanic adds a dynamic and visually engaging element to the gameplay. The propeller is a special tile that, when activated, moves across the board to clear a specific tile. Implementing this mechanic involved utilizing Bezier curves to define smooth, curved paths for the propeller's movement. This approach allowed for more natural and fluid animations, enhancing the overall visual quality and player experience. The propeller’s path can vary depending on its initial position and the target tiles, making its behavior unpredictable and exciting for players. There are also 2 modes to implement that mechanic:

1. Same target mode: All propellers targets the same cell. If target is blasted, remaining propellers choose new target and new bezier curves are created (one of the most challenging parts of this project).

2. Different target mode: Propellers do not intend to choose same target, so they just fly any direction they wanted

Mode can easily change in Unity Inspector:

![3 mode-change](https://github.com/ensargoktekin/CENG469-INFO/assets/93570150/bf9034f2-0bc4-4fdd-ab34-2b37518a397b)

The Bezier curves provided a robust method to calculate intermediate points along the propeller’s path, ensuring that its movement is both precise and aesthetically pleasing. This implementation required careful tuning of the control points to achieve the desired movement patterns and ensure that the propeller effectively interacts with other game elements. There is a Route Manager to monitor all Bezier Curve actions.

Creating a regular route in Route Manager:
```c#
private Route CreateRegularRoute(Transform start, Transform end)
{
    Route route = Instantiate(_routePrefab);
    route.transform.SetParent(_gridView);

    int random = UnityEngine.Random.RandomRange(-15, 16);
    random *= 2;
    //Debug.Log(random);
    Vector3 p2Vect = Quaternion.Euler(0, 0, random * 3) * (end.position - start.position) / 4;
    Vector3 p3Vect = Quaternion.Euler(0, 0, random) * (end.position - start.position) * 3 / 4;

    route.InitRoute(start.position, p2Vect + start.position, p3Vect + start.position, end.position, _targetPos);
    return route;
}
```
Initializing a new route or updating the current route in Route.cs
```c#
public void InitRoute(Vector3 p1pos, Vector3 p2pos, Vector3 p3pos, Vector3 p4pos, Vector2 end)
{
    P1pos = p1pos;
    P2pos = p2pos;
    P3pos = p3pos;
    P4pos = p4pos;
    End = end;
}

public void UpdateRoute(Transform start, Transform end, Vector2 endCoord, Vector3 initDirection)
{
    int random = UnityEngine.Random.RandomRange(-15, 16);
    random *= 2;
    Vector3 p2Vect = Quaternion.Euler(0, 0, random * 3) * (initDirection * 5 - start.position) / 4;
    Vector3 p3Vect = Quaternion.Euler(0, 0, random) * (end.position - start.position) * 3 / 4;

    P1pos = start.position;
    P2pos = P1pos + p2Vect;
    P3pos = P1pos + p3Vect;
    P4pos = end.position;
    End = endCoord;
}
```
Getting the current direction in a route following object to calculate new bezier bezier curve and to visualize current direction:
```c#
Vector3 GetDirection(float _tParam, Vector3 p1, Vector3 p2, Vector3 p3, Vector3 p4)
{
    // Calculate the first derivative of the Bezier curve
    Vector3 direction = 3 * Mathf.Pow(1 - _tParam, 2) * (p2 - p1) +
                        6 * (1 - _tParam) * _tParam * (p3 - p2) +
                        3 * Mathf.Pow(_tParam, 2) * (p4 - p3);

    // Normalize the direction vector to get a unit vector
    return direction.normalized;
}
```
Direction vector is indicated as green:

![4 direction](https://github.com/ensargoktekin/CENG469-INFO/assets/93570150/364ee410-77aa-4120-afa9-945eb420815b)

Drawing path for a route with Gizmos
```c#
private void OnDrawGizmos()
{
    // Bezier curve points:
    // P = (1 - t)^3 * P1 +
    // 3 * (1 - t^2) * t * P2 +
    // 3 * (1 - t) * t^2 * P3 +
    // t^3 * P4
    Gizmos.color = Color.blue;
    for (float t = 0; t <= 1; t += 0.01f)
    {
        _gizmosPos = Mathf.Pow(1 - t, 3) * P1pos +
            3 * Mathf.Pow(1 - t, 2) * t * P2pos +
            3 * (1 - t) * Mathf.Pow(t, 2) * P3pos +
            Mathf.Pow(t, 3) * P4pos;

        Gizmos.DrawSphere(_gizmosPos, 0.01f);
    }

    Gizmos.color = Color.red;

    Gizmos.DrawLine(new Vector3(P1pos.x, P1pos.y, P1pos.z),
                    new Vector3(P2pos.x, P2pos.y, P2pos.z));

    Gizmos.DrawLine(new Vector3(P3pos.x, P3pos.y, P3pos.z),
                    new Vector3(P4pos.x, P4pos.y, P4pos.z));
}
```
As seen in the code above, path is indicated as blue dots and vectors (P1-P2 and P3-P4) are indicated as red:

![5 Gizmos](https://github.com/ensargoktekin/CENG469-INFO/assets/93570150/5e822372-e5b3-4992-b6ad-c03fdb1fa53a)

## Runtime Videos

Runtime video of match3 features: [link](https://www.youtube.com/watch?v=2I-AFx7nFDE)

Runtime video of project (mainly focus on bezier part): [link](https://www.youtube.com/watch?v=_sLCin3Zs6E)

