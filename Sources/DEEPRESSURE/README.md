# DEEPRESSURE

![](/Sources/DEEPRESSURE/Images/LightsShaders.gif)    |  ![](/Sources/DEEPRESSURE/Images/Thumbnail-Deepressure.png)
:-------------------------:|:-------------------------:
 ![](/Sources/DEEPRESSURE/Images/BoidsSystem-ezgif.com-resize.gif) |  ![](/Sources/DEEPRESSURE/Images/Window_Water_Shader.gif)


 <details>
  <Summary>CODE: Boids system</Summary>

  
  ```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

//====================================================|=============|========================================\\
//====================================================|=============|=========================================\\
//----------------------------------------------------|-FISH OBJECT-|----------------------------------------\\
//====================================================|=============|==========================================\\
//====================================================|=============|===========================================\\

[System.Serializable]
public class FishObject {
    //--->
    public string FishGroupName { get { return fishGroupName; } }
    public GameObject ObjPrefab { get { return prefab; } }
    public bool EnableSpawner { get { return enableSpawner; } }
    //---|

    [Header("AI Group Stats")]
    [SerializeField] private string fishGroupName;
    [SerializeField] private GameObject prefab;
    [SerializeField] private bool enableSpawner;

    [Header("Fish Settings")]
    [Range(1, 100)]  public int numOfFish;
    [Range(0, 5)] public float minSpeed;
    [Range(0, 5)] public float maxSpeed;
    [Range(1, 5)] public float rotationSpeed;
    [Range(1, 20)] public float neighbourDistance;
    public Vector3 goalPos;
}

//====================================================|===============|=====================================\\
//====================================================|===============|=======================================\\
//----------------------------------------------------|-BOIDS MANAGER-|----------------------------------------\\
//====================================================|===============|=========================================\\
//====================================================|===============|==========================================\\

public class BoidsManager : MonoBehaviour {
    public static BoidsManager instance;
    
    [Header("Main Settings")]
    public Vector3 swimArea = new Vector3(20, 20, 20);
    [HideInInspector]public List<GameObject> allFish = new List<GameObject>();
    public Vector3 playersPos;

    [Header("Dangerous Creatures")]
    public List<GameObject> dangerousCreatures;

    public Vector3 goalPos;

    [Header("Fishes")]
    public List<FishObject> fishes;

    [Header("Checkers")]
    public bool isOnSight = true;
    public float dangerDistance = 10f;
    public float spawnCheckRadius = 1;

    private void Awake() {
        if(instance == null) instance = this;    
    }

    void Start() {
        foreach (var fishType in fishes) {
            var newFish = new GameObject(fishType.FishGroupName);
            if (fishType.ObjPrefab != null && fishType.EnableSpawner) {
                for (int i = 0; i < fishType.numOfFish; i++) {
                    Vector3 pos = GetAvailableSpawnPosition();
                    var fish  = Instantiate(fishType.ObjPrefab, pos, Quaternion.identity);
                    fish.AddComponent<FishInfo>().fishType = fishType;
                    allFish.Add(fish);
                    allFish[i].gameObject.transform.parent = newFish.transform;
                }
                goalPos = transform.position;
            }
        }
    }

    private void Update() {
        playersPos = GameManager.Instance.player.transform.position;

        if (Random.Range(0, 400) < 10) {
            goalPos = GetNewGoalPosition();
        }

        foreach (GameObject fish in allFish) {
            Flock flock = fish.GetComponent<Flock>();
            var isNearDanger = IsNearDanger(fish.transform.position);
            if (isNearDanger) {
                flock.isScattering = true;
            }
            else {
                flock.isScattering = false;
            }
        }
    }

    private Vector3 GetAvailableSpawnPosition() {
        Vector3 pos;
        bool positionFound = false;

        int maxAttempts = 100;  // Limit to avoid infinite loops
        int attempts = 0;

        do {
            pos = transform.position + new Vector3(
                Random.Range(-swimArea.x, swimArea.x),
                Random.Range(-swimArea.y, swimArea.y),
                Random.Range(-swimArea.z, swimArea.z)
            );

            // Check if there aren't obstacles
            if (!Physics.CheckSphere(pos, spawnCheckRadius)) {
                // Also, make sure Fish dont swim(Spawn) inside the player 
                if (Vector3.Distance(pos, playersPos) > spawnCheckRadius) {
                    positionFound = true;
                }
            }

            attempts++;
        } while (!positionFound && attempts < maxAttempts);

        return pos;
    }

    private Vector3 GetNewGoalPosition() {
        var swimableArea = swimArea - playersPos;
        return transform.position + new Vector3(Random.Range(-swimableArea.x, swimableArea.x),
            Random.Range(-swimableArea.y, swimableArea.y),
            Random.Range(-swimableArea.z, swimableArea.z));
    }

    public void ActivateVisuals(bool isActive = false) {
        for (int i = 0; i < allFish.Count; i++) {
            allFish[i].SetActive(isActive);
        }
    }

    private bool IsNearDanger(Vector3 fishPos) {
        // Check distance from the player
        if (Vector3.Distance(fishPos, playersPos) < dangerDistance) {
            return true;
        }

        // Check distance from each dangerous creature
        foreach (GameObject danger in dangerousCreatures) {
            if (Vector3.Distance(fishPos, danger.transform.position) < dangerDistance) {
                return true;
            }
        }

        return false;
    }

    private void OnDrawGizmosSelected() {
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireCube(transform.position, swimArea * 2);
    }
}
  ```

 </details>
