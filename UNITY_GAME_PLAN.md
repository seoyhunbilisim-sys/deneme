# Mobil Oyun Prototipi — “Şerit Değiştiren Dodger”

Bu doküman, 3 şeritli, tek elle oynanabilen, kısa oturumlu (30–90 sn) ve **zorluk ramp** içeren mobil oyunun temel prototip tasarımını ve Unity sahne/skript kurulumunu içerir.

---

## 1) Game Design Doc (1 sayfa)

### Oyun Tanımı
- **Tür:** Endless dodger / refleks.
- **Temel Mekanik:** Oyuncu 3 şerit arasında ilerler; ekrana dokununca **bir şerit sağ/sol** gider. Engellerden kaçınır, Türkiye temalı “collectible”ları toplar.
- **Kontrol (temiz yaklaşım):** **Sol üst** dokunuş = sola geçiş, **sağ üst** dokunuş = sağa geçiş. (Alternatif: swipe sağ/sol; prototipte tap tercih.)

### Oynanış Döngüsü
1. Başlangıç (MainMenu) → Start.
2. Oyun başlar, engeller akmaya başlar.
3. Oyuncu şerit değiştirerek hayatta kalır ve collectible toplar.
4. Zorluk zamanla artar.
5. Çarpışma → Game Over → Restart/Home.

### Zorluk Eğrisi (Difficulty Ramp)
- **Her 10 saniyede bir:**
  - Oyun hızı artar.
  - Engel spawn oranı artar.
  - Collectible spawn oranı düşer.

### Skor Sistemi
- **Skor = hayatta kalma süresi + collectible puanı.**
  - Süre puanı: geçen saniye başına 1.
  - Collectible puanı: her collectible 5 (özelleştirilebilir).
- Best score PlayerPrefs ile saklanır.

### Türkiye Teması (Minimal)
- Arka plan: düz vektör/gradient üzerinde **İstanbul silüeti + Boğaz köprüsü**.
- Collectible’lar (3–5): **simit, çay bardağı, nazar boncuğu, lale, vapur**.
  - Tamamı basit/özgün placeholder sprite (sonradan değişebilir).

### UI Akışı
```
Boot → MainMenu → Game → GameOver
                ↘ Settings
```

---

## 2) Unity Sahne Planı

### Scenes
- **Boot**: Localization init tamamlanır, sonra MainMenu yüklenir.
- **MainMenu**: Start, Language, Settings.
- **Game**: Core gameplay, score, pause.
- **GameOver**: Skor, Best, Restart, Home.
- **Settings**: Sound/Vibration, Language.

---

## 3) Dosya/Klasör Yapısı (Öneri)
```
Assets/
  Scenes/
    Boot.unity
    MainMenu.unity
    Game.unity
    GameOver.unity
    Settings.unity
  Scripts/
    Core/
      GameManager.cs
      DifficultyManager.cs
      ScoreManager.cs
      LocalizationManager.cs
      BootLoader.cs
      ScoreSnapshot.cs
    Gameplay/
      PlayerController.cs
      Spawner.cs
      Mover.cs
      HitDetector.cs
    UI/
      GameOverUI.cs
  UI/
    Prefabs/ (butonlar, paneller)
  Art/
    Sprites/ (placeholder)
  Localization/
    Tables/ (String Tables)
```

---

## 4) Script’ler (Kopyala-Yapıştır Hazır)

> Aşağıdaki script’ler Unity 2022/2023 LTS uyumludur. **Her script’in başında kısa açıklama bulunur.**

### `PlayerController.cs`
```csharp
using UnityEngine;

// 3 şeritli oyuncu kontrolü: sol/sağ üst dokunuş ile bir şerit kaydırır.
public class PlayerController : MonoBehaviour
{
    [SerializeField] private float laneOffset = 2f;
    [SerializeField] private float moveSpeed = 10f;
    [SerializeField] private int currentLane = 1; // 0=sol,1=orta,2=sağ

    private Vector3 targetPos;

    private void Start()
    {
        targetPos = transform.position;
    }

    private void Update()
    {
        HandleInput();
        MoveToLane();
    }

    private void HandleInput()
    {
        if (Input.GetMouseButtonDown(0))
        {
            Vector3 pos = Input.mousePosition;
            bool isLeft = pos.x < Screen.width / 2f;
            int dir = isLeft ? -1 : 1;
            ChangeLane(dir);
        }
    }

    private void ChangeLane(int direction)
    {
        currentLane = Mathf.Clamp(currentLane + direction, 0, 2);
        float x = (currentLane - 1) * laneOffset;
        targetPos = new Vector3(x, transform.position.y, transform.position.z);
    }

    private void MoveToLane()
    {
        transform.position = Vector3.Lerp(transform.position, targetPos, moveSpeed * Time.deltaTime);
    }
}
```

### `Spawner.cs`
```csharp
using UnityEngine;

// Engel ve collectible üretimi (oranlar DifficultyManager tarafından güncellenir).
// Aynı şeritte arka arkaya obstacle spam engellenir.
public class Spawner : MonoBehaviour
{
    [SerializeField] private GameObject[] obstacles;
    [SerializeField] private GameObject[] collectibles;
    [SerializeField] private float spawnInterval = 1.2f;
    [SerializeField] private float collectibleChance = 0.3f;
    [SerializeField] private float laneOffset = 2f;

    private float timer;
    private int lastObstacleLane = -1;

    private void Update()
    {
        timer += Time.deltaTime;
        if (timer >= spawnInterval)
        {
            timer = 0f;
            Spawn();
        }
    }

    public void SetSpawnParams(float newInterval, float newCollectibleChance)
    {
        spawnInterval = newInterval;
        collectibleChance = newCollectibleChance;
    }

    private void Spawn()
    {
        bool spawnCollectible = Random.value < collectibleChance;
        int lane = Random.Range(0, 3);
        if (!spawnCollectible && lastObstacleLane == lane)
        {
            lane = (lane + Random.Range(1, 3)) % 3;
        }

        float x = (lane - 1) * laneOffset;
        Vector3 spawnPos = new Vector3(x, transform.position.y, transform.position.z);

        if (spawnCollectible && collectibles.Length > 0)
        {
            Instantiate(collectibles[Random.Range(0, collectibles.Length)], spawnPos, Quaternion.identity);
        }
        else if (obstacles.Length > 0)
        {
            Instantiate(obstacles[Random.Range(0, obstacles.Length)], spawnPos, Quaternion.identity);
            lastObstacleLane = lane;
        }
    }
}
```

### `DifficultyManager.cs`
```csharp
using UnityEngine;

// Her 10 saniyede bir hız/spawn oranı artırır ve collectible oranını azaltır.
public class DifficultyManager : MonoBehaviour
{
    [SerializeField] private float interval = 10f;
    [SerializeField] private float speedIncrement = 0.5f;
    [SerializeField] private float spawnIntervalDecrement = 0.1f;
    [SerializeField] private float collectibleChanceDecrement = 0.05f;

    [SerializeField] private float minSpawnInterval = 0.4f;
    [SerializeField] private float minCollectibleChance = 0.05f;

    [SerializeField] private Spawner spawner;
    [SerializeField] private float currentSpawnInterval = 1.2f;
    [SerializeField] private float currentCollectibleChance = 0.3f;
    [SerializeField] private float gameSpeed = 5f;

    private float timer;

    public float GameSpeed => gameSpeed;

    private void Update()
    {
        timer += Time.deltaTime;
        if (timer >= interval)
        {
            timer = 0f;
            IncreaseDifficulty();
        }
    }

    private void IncreaseDifficulty()
    {
        gameSpeed += speedIncrement;
        currentSpawnInterval = Mathf.Max(minSpawnInterval, currentSpawnInterval - spawnIntervalDecrement);
        currentCollectibleChance = Mathf.Max(minCollectibleChance, currentCollectibleChance - collectibleChanceDecrement);
        spawner.SetSpawnParams(currentSpawnInterval, currentCollectibleChance);
    }
}
```

### `Mover.cs`
```csharp
using UnityEngine;

// Engel ve collectible'ları aşağı doğru akar; hız DifficultyManager'dan alınır.
public class Mover : MonoBehaviour
{
    [SerializeField] private float destroyY = -7f;
    [SerializeField] private DifficultyManager difficultyManager;

    private void Update()
    {
        if (difficultyManager == null) return;
        float speed = difficultyManager.GameSpeed;
        transform.Translate(Vector3.down * speed * Time.deltaTime);

        if (transform.position.y < destroyY)
        {
            Destroy(gameObject);
        }
    }
}
```

### `ScoreManager.cs`
```csharp
using UnityEngine;
using TMPro;

// Skor ve best score hesaplar (hayatta kalma süresi + collectible puanı).
public class ScoreManager : MonoBehaviour
{
    [SerializeField] private TMP_Text scoreText;
    [SerializeField] private int collectiblePoints = 5;

    private float survivalTime;
    private int collectedScore;

    private void Update()
    {
        survivalTime += Time.deltaTime;
        UpdateUI();
    }

    public void AddCollectible()
    {
        collectedScore += collectiblePoints;
    }

    public int GetTotalScore()
    {
        return Mathf.FloorToInt(survivalTime) + collectedScore;
    }

    private void UpdateUI()
    {
        if (scoreText != null)
        {
            scoreText.text = GetTotalScore().ToString();
        }
    }
}
```

### `GameManager.cs`
```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

// Oyun state yönetimi: Start, GameOver, Restart.
public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }

    [SerializeField] private ScoreManager scoreManager;
    [SerializeField] private DifficultyManager difficultyManager;
    [SerializeField] private GameObject pausePanel;

    private void Awake()
    {
        if (Instance == null) Instance = this;
        else Destroy(gameObject);

        Application.targetFrameRate = 60;
        Screen.sleepTimeout = SleepTimeout.NeverSleep;
    }

    public void GameOver()
    {
        ScoreSnapshot.Capture(scoreManager.GetTotalScore());
        SceneManager.LoadScene("GameOver");
    }

    public void Restart()
    {
        SceneManager.LoadScene("Game");
    }

    public void GoHome()
    {
        SceneManager.LoadScene("MainMenu");
    }

    public void Pause()
    {
        Time.timeScale = 0f;
        if (pausePanel != null) pausePanel.SetActive(true);
    }

    public void Resume()
    {
        Time.timeScale = 1f;
        if (pausePanel != null) pausePanel.SetActive(false);
    }
}
```

### `ScoreSnapshot.cs`
```csharp
using UnityEngine;

// Game sahnesinden GameOver sahnesine skor taşır ve best score hesaplar.
public static class ScoreSnapshot
{
    private const string BestScoreKey = "BestScore";

    public static int LastScore { get; private set; }
    public static int BestScore { get; private set; }

    public static void Capture(int totalScore)
    {
        LastScore = totalScore;
        int best = PlayerPrefs.GetInt(BestScoreKey, 0);
        if (totalScore > best)
        {
            best = totalScore;
            PlayerPrefs.SetInt(BestScoreKey, best);
        }
        BestScore = best;
    }
}
```

### `GameOverUI.cs`
```csharp
using TMPro;
using UnityEngine;

// GameOver sahnesinde skor ve best skor yazdırır.
public class GameOverUI : MonoBehaviour
{
    [SerializeField] private TMP_Text scoreText;
    [SerializeField] private TMP_Text bestText;

    private void Start()
    {
        if (scoreText != null) scoreText.text = ScoreSnapshot.LastScore.ToString();
        if (bestText != null) bestText.text = ScoreSnapshot.BestScore.ToString();
    }
}
```

### `HitDetector.cs`
```csharp
using UnityEngine;

// Obstacle ve collectible temaslarını yönetir.
public class HitDetector : MonoBehaviour
{
    [SerializeField] private ScoreManager scoreManager;

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (other.CompareTag("Obstacle"))
        {
            GameManager.Instance.GameOver();
        }
        else if (other.CompareTag("Collectible"))
        {
            scoreManager.AddCollectible();
            Destroy(other.gameObject);
        }
    }
}
```
### `LocalizationManager.cs`
```csharp
using UnityEngine;
using UnityEngine.Localization.Settings;

// Dil seçimi ve PlayerPrefs kaydı (tr/en/ar).
public class LocalizationManager : MonoBehaviour
{
    private const string LangKey = "LangCode";

    private async void Start()
    {
        await LocalizationSettings.InitializationOperation.Task;
        string code = PlayerPrefs.GetString(LangKey, "tr");
        SetLanguage(code);
    }

    public void SetLanguage(string code)
    {
        var locales = LocalizationSettings.AvailableLocales.Locales;
        foreach (var locale in locales)
        {
            if (locale.Identifier.Code == code)
            {
                LocalizationSettings.SelectedLocale = locale;
                PlayerPrefs.SetString(LangKey, code);
                break;
            }
        }
    }
}
```

### `BootLoader.cs`
```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.Localization.Settings;

// Boot sahnesi açılışında localization hazır olunca MainMenu yükler.
public class BootLoader : MonoBehaviour
{
    private async void Start()
    {
        await LocalizationSettings.InitializationOperation.Task;
        SceneManager.LoadScene("MainMenu");
    }
}
```

---

## 5) UI Hiyerarşisi (Öneri)

### MainMenu
```
Canvas
  Title
  Button_Start
  Button_Settings
  LanguagePanel
    Button_TR
    Button_EN
    Button_AR
```

### Settings
```
Canvas
  Toggle_Sound
  Toggle_Vibration
  Button_Back
```

### Game
```
Canvas
  TMP_Text_Score
  Button_Pause
  Panel_Pause (inactive)
    Button_Resume
    Button_Home
```

### GameOver
```
Canvas
  TMP_Text_Score
  TMP_Text_Best
  Button_Restart
  Button_Home
```

---

## 6) Prefab Ayarları (Collider/Rigidbody2D)

### Player
- **Collider2D:** BoxCollider2D (isTrigger = true).
- **Rigidbody2D:** Kinematic.
- **Tag:** Player.
- **HitDetector** script’i Player üzerinde.

### Obstacle
- **Collider2D:** BoxCollider2D (isTrigger = true).
- **Rigidbody2D:** Kinematic.
- **Tag:** Obstacle.
- **Mover** script’i Obstacle üzerinde, DifficultyManager referansı atanmış.

### Collectible
- **Collider2D:** CircleCollider2D (isTrigger = true).
- **Rigidbody2D:** Kinematic.
- **Tag:** Collectible.
- **Mover** script’i Collectible üzerinde.

---

## 7) Sahne Hiyerarşisi ve UI Bağlama

### Game Scene
```
Game
  MainCamera (orthographic)
  GameManager (GameManager, DifficultyManager, ScoreManager refs)
  Spawner (spawn point: kameranın üstünde)
  Player (PlayerController + HitDetector)
  Canvas
    TMP_Text_Score (TMP_Text → ScoreManager)
    Button_Pause (OnClick → GameManager.Pause)
    Panel_Pause (inactive)
      Button_Resume (OnClick → GameManager.Resume)
      Button_Home (OnClick → GameManager.GoHome)
```

### GameOver Scene
```
GameOver
  Canvas
    TMP_Text_Score (TMP_Text → GameOverUI.scoreText)
    TMP_Text_Best (TMP_Text → GameOverUI.bestText)
    Button_Restart (GameManager.Restart)
    Button_Home (GameManager.GoHome)
```

### Settings Scene
```
Settings
  Canvas
    Button_TR → LocalizationManager.SetLanguage("tr")
    Button_EN → LocalizationManager.SetLanguage("en")
    Button_AR → LocalizationManager.SetLanguage("ar")
```

### Boot Scene
```
Boot
  LocalizationManager (async init)
  BootLoader (MainMenu yükler)
```

---

## 8) Asset İhtiyacı (Placeholder)
- **Oyuncu:** Basit daire/üçgen.
- **Engeller:** Kare/dikdörtgen.
- **Collectible:** Simit, çay bardağı, nazar, lale, vapur (basit ikonlar).
- **Arka plan:** İstanbul/Boğaz silüeti (düz vektör).

---

## 9) Android Build Checklist

### Player Settings
- Company Name / Product Name ayarla.
- Bundle Identifier (ör: `com.studio.lanedodger`).
- Version: `1.0.0`, Bundle Version Code: `1`.
- Target API: **Android 33+** (Google Play gereksinimi).
- Scripting Backend: **IL2CPP**.
- ARM64 build aktif.
- Minimum API: Android 7.0 (API 24) veya üstü.

### Performans
- 60 FPS hedefi (Application.targetFrameRate = 60) GameManager.Awake içinde.
- Fixed timestep 0.02.
- Sprite atlas/texture compression.
- Screen.sleepTimeout = NeverSleep.
- Input: tek dokunuş (tap) ve UI için EventSystem aktif.

### Google Play Yayın Notları
- Privacy Policy URL.
- App content rating.
- Keystore imzalama.
- Versioning düzenli artır.

---

## 10) Polish Önerileri (Sonraki Adım)
- Partikül efektleri (çarpışma, collectible).
- Haptic feedback (vibration toggle).
- Minimal UI animasyonları.
- Günlük görevler/başarımlar.
```
