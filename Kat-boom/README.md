#                                                             Kat-boom


![](/Kat-boom/Images/GhostMode.gif)    |  ![](/Kat-boom/Images/Movement.gif)
:-------------------------:|:-------------------------:

## Summary of the project
Kat-boom was a 4-day game project that was my first game project with a team.

[Kat-boom ITCH.IO](https://ebben.itch.io/katboom)
## About

Kat-boom is a top-down puzzle. Where you play as a ghost cat hunting her beloved tarn ball
To be able to rest in peace. Use the ability to go through walls and destroy all the barrels to find the yarn ball
you know be a cat!

### -  Player Movement
In kat-boom, the movement is grid-based, which means the cat can only move from one square to another without having the ability to 
move diagonally.


<details>
  <summary>PlayerManager</summary>

```cs
public class PlayerController : MonoBehaviour
{

    public GameObject ghostEffect;
    public float speed = 25f;
    public Transform movePoint;

    public LayerMask whatStops;
    public LayerMask neverGoThrough;
    private LayerMask tempLayer;


    public bool isTransparent = false;
    private bool canUseGM = true;
    public float ghostMeterTimer = 0.5f;
    public float GMamount = 1;

    private Color origColor;
    bool isRight = true;

    public int lives = 9;

    public bool canMove = false;

    public SpriteRenderer spriteRenderer;
    new Collider2D collider;

    public float x;
    public float y;

    
    void Start()
    {
        canUseGM = GameManager.Instance.GMready;
        movePoint.parent = null;

        spriteRenderer = GetComponentInChildren<SpriteRenderer>();
        collider = GetComponent<Collider2D>();

        spriteRenderer.color = Color.white;
        origColor = spriteRenderer.color;
        tempLayer = whatStops;
    }

    void Update()
    {
        if (!canMove)
            return;

        x = Input.GetAxisRaw("Horizontal");
        y = Input.GetAxisRaw("Vertical");

        //Move 
        transform.position = Vector3.MoveTowards(transform.position, movePoint.position, speed * Time.deltaTime);
        if (Vector3.Distance(transform.position, movePoint.position) == 0)
        {
            if (Mathf.Abs(x) == 1)
            {
                if (!Physics2D.OverlapCircle(movePoint.position + new Vector3(x, 0, 0), 0.2f, whatStops))
                {
                    movePoint.position += new Vector3(x, 0, 0);
                    if (x > 0 && !isRight)
                        FlipHori();
                    else if (x < 0 && isRight)
                        FlipHori();
                }
            }
            else if (Mathf.Abs(y) == 1)
            {
                if (!Physics2D.OverlapCircle(movePoint.position + new Vector3(0, y, 0), 0.2f, whatStops))
                {
                    movePoint.position += new Vector3(0, y, 0);
                }
            }
        }

        if (Input.GetKeyDown(KeyCode.Space) && canUseGM)
        {
            StartCoroutine(Transparent());
        }
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        //movePoint.position = transform.position;
        if(other.gameObject.layer == neverGoThrough)
            StartCoroutine(GameManager.Instance.Death());

        if (other.gameObject == GameManager.Instance.door1)
            GameManager.Instance.levelSystem.NxtLvl(1,2);

        if (other.gameObject == GameManager.Instance.door2)
            GameManager.Instance.levelSystem.NxtLvl(2,3);

        if(other.gameObject.tag == "explosion")
            StartCoroutine(GameManager.Instance.Death());

        if(other.gameObject.tag == "Enemy")
        {
            GameManager.Instance.batHurt = other.gameObject.GetComponent<AudioSource>();
            GameManager.Instance.batHurt.Play();
        }
        if(other.gameObject.tag == "El")
        {
            GameManager.Instance.elHurt = other.gameObject.GetComponent<AudioSource>();
            GameManager.Instance.elHurt.Play();
        }
        if (other.gameObject.tag == "DiaTrigger")
            GameManager.Instance.lastDia.TriggerDia();

        if (other.gameObject.tag == "Goal")
            GameManager.Instance.EndScreen("Victory");

    }

    public IEnumerator Transparent()
    {
        if(!isTransparent)
        {
            canUseGM = false;
            isTransparent = true;

            GMamount = 0;

            Color newColor = spriteRenderer.color;
            newColor.a = 0.3f;
            spriteRenderer.color = newColor;

            collider.isTrigger = true;

            whatStops &= LayerMask.GetMask("Walls");

            yield return new WaitForSeconds(ghostMeterTimer);

            isTransparent = false;
            collider.isTrigger = false;
            spriteRenderer.color = origColor;
            whatStops = tempLayer;
            yield return new WaitForSeconds(ghostMeterTimer * 6);

            var gE = Instantiate(ghostEffect, transform.position, Quaternion.identity);
            Destroy(gE, .6f);
            //GMamount += 1;
            canUseGM = true;

        }
    }

    void FlipHori()
    {
        var currScale = transform.localScale;
        currScale.x *= -1;
        transform.localScale = currScale;
        isRight = !isRight;
    }

    public void TakeDamage() {
        GameManager.Instance.hearts[lives - 1].fillAmount = 0;
        lives--;
    }
    public void Teleport(Vector3 position)
    {
        transform.position = position;
        movePoint.position = transform.position;
    }

}
```
</details>

<details>
  <summary> GameManager </summary>

```cs
public class GameManager : MonoBehaviour
{
    //SINGELTON
    public static GameManager Instance;

    #region Refrenceses
    [Header("--REFS--")]
    public PlayerController player;
    public GameObject explosion;
    public GameObject DoorElL1;
    public GameObject DoorElL2;
    public GameObject DoorElL3;
    public GameObject DeadCat;
    public Leaderboard leaderboard;
    public GameObject victoryScreen;
    public GameObject Gaol;
    public DiaTrigger lastDia;
    private DiaTrigger diafst;
    public LevelSystem levelSystem;
    #endregion

    public bool isDead = false;

    [Header("--Boxes Count--")]
    public int countBoxesLvl1 = 5;
    public int countBoxesLvl2 = 5;
    public int countBoxesLvl3 = 5;
    public int countBoxesLvl0 = 1;

    [Header("--REF Doors--")]
    public GameObject door1;
    public GameObject door2;


    [Header("--UI MANAGEMENT--")]
    public int score = 0;

    [Header("--UI--")]
    public TextMeshProUGUI scoreTxt;
    public TextMeshProUGUI livesTxt;
    public Image ghostMeeterFill;
    public Image[] hearts = new Image[9];
    public GameObject gameOverscrn;
    public GameObject HUD;

    [Header("--GAMEOVERUI--")]
    public TextMeshProUGUI title;

    [Header("--AUDIO--")]
    public AudioSource doorSfx;
    public AudioSource doorSfx2;
    public AudioSource elHurt; 
    public AudioSource batHurt;



    public bool GMready { get { return ghostMeeterFill.fillAmount >= 1; } }

    private void Awake()
    {
        Instance = this;
    }

    // Start is called before the first frame update
    void Start()
    {
        levelSystem = GetComponent<LevelSystem>();
        diafst = GetComponent<DiaTrigger>();
        Invoke("DiaPlay", 0.01f);
        HUD.SetActive(true);
        gameOverscrn.SetActive(false);
        victoryScreen.SetActive(false);

        door1.SetActive(false);
        door2.SetActive(false);
        Gaol.SetActive(false);
        
    }


    void Update()
    {
        //UI
        UIUpdate();

        CheckingForLeveling();

        if (player.lives > 0 && isDead)
        {
            StartCoroutine(Death());
            isDead = false;
        }

        if (!GMready)
        {
            ghostMeeterFill.fillAmount = Mathf.MoveTowards(ghostMeeterFill.fillAmount, 1, Time.deltaTime);
            player.GMamount = Mathf.MoveTowards(player.GMamount, 1, Time.deltaTime * 0.26f);
        }

    }

    public void CheckingForLeveling()
    {
        if (countBoxesLvl0 < 1 && levelSystem.tut)
        {
            levelSystem.NxtLvl(0, 1);
        }
        if (countBoxesLvl1 < 1 && levelSystem.lvl1)
        {
            door1.SetActive(true);
            doorSfx.Play();

            DoorElL1.SetActive(false);
        }

        if (countBoxesLvl2 < 1 && levelSystem.lvl2)
        {
            door2.SetActive(true);
            doorSfx.Play();

            DoorElL2.SetActive(false);

        }
        if (countBoxesLvl3 < 1 && levelSystem.lvl3)
        {
            Gaol.SetActive(true);

        }
    }

    private void UIUpdate()
    {
        ghostMeeterFill.fillAmount = player.GMamount;
        scoreTxt.text = "Score: " + score.ToString();
    }



    public IEnumerator Death()
    {
        player.spriteRenderer.enabled = false;
        player.canMove = false;
        var deadCat = Instantiate(DeadCat, player.transform.position, Quaternion.identity);
        levelSystem.SwitchingLevels();


        Destroy(deadCat, .7f);

        yield return new WaitForSeconds(.8f);

        player.spriteRenderer.enabled = true;
        hearts[player.lives - 1].fillAmount = 0;
        player.lives--;
        score -= 25;

        yield return new WaitForSeconds(1f);

        if (player.lives <= 0)
        {
            player.canMove = false;

            player.lives = 0;
            EndScreen("GameOver");
        }else
            player.canMove = true;

    }


    public void EndScreen(string text)
    {
        StartCoroutine(leaderboard.SubmitScoreRoutine(score));
        player.canMove = false;
        title.text = text;
        gameOverscrn.SetActive(true);
        HUD.SetActive(false);
        if (text == "Victory")
        {
            score += 100;
            victoryScreen.SetActive(true);
        }
        else
            victoryScreen.SetActive(false);

    }

    void DiaPlay()
    {
        diafst.TriggerDia();
    }

    public void Rstrt()
    {
        SceneManager.LoadScene(1);
        Time.timeScale = Time.timeScale == 0 ? 1 : 1;
    }

    public void Menu()
    {
        SceneManager.LoadScene(0);

    }
}
```
</details>
