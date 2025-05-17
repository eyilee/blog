---
title: MonoGame 範例 NeonShooter 之 3
categories: MonoGame
date: 2025-05-11 20:44:29
updated: 2025-05-11 20:44:29
tags:
---

這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第三篇，本篇將完成以下部分：
1. 敵人生成器
2. 碰撞處理

<!-- more -->
### 1. 敵人生成器
如同角色和子彈，先新增一個 Enemy.cs，宣告一個 class `Enemy`，加入必要的函式。

{% codeblock Enemy.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

namespace NeonShooter
{
    class Enemy
    {
        private Texture2D m_Image;
        private Vector2 m_Position = Vector2.Zero;
        private Vector2 m_Velocity = Vector2.Zero;
        private Color m_Color = Color.White;
        private float m_Rotation = 0f;
        private Vector2 m_Size = Vector2.Zero;
        private float m_Scale = 1f;
        private bool m_IsExpired = false;

        public Vector2 Position { get { return m_Position; } }
        public float Rotation { get { return m_Rotation; } }
        public Vector2 Size { get { return m_Size; } }
        public bool IsExpired { get { return m_IsExpired; } }

        public Enemy (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
        {
            m_Image = _image;
            m_Position = _position;
            m_Velocity = _velocity;
            m_Rotation = _rotation;
            m_Size = new Vector2 (_image.Width, _image.Height);
        }

        public void Update ()
        {
        }

        public void Draw (SpriteBatch _spriteBatch)
        {
            _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
        }
    }
}
{% endcodeblock %}

`Enemy` 暫時只有基本的功能，接著新增 EnemySpawner.cs，宣告一個 class `EnemySpawner` 作為敵人生成器，`EnemySpawner` 將用來決定何時生成敵人，因為遊戲中只需要一個生成器所以宣告成 static。

> 為了說明方便，之後每一次 `Update` 將以每一個 frame 或每一幀來描述。

範例中使用了一個變數 `inverseSpawnChance` 用來決定生成敵人的機率，每一幀以 `inverseSpawnChance` 為參數呼叫一次 `Random.Next`，將會獲得一個 0 到 `inverseSpawnChance` 之間但不包含 `inverseSpawnChance` 的隨機數。

為了使用 `Random.Next` 所以需要宣告一個 `Random` 變數 `m_Random`，再宣告一個 `float` 變數 `m_nInverseSpawnChance`，初始值設為 90 代表遊戲開始時每一幀生成敵人的機率是 $\frac{1}{90}$。

{% codeblock EnemySpawner.cs lang:csharp %}
using Microsoft.Xna.Framework;
using System;

namespace NeonShooter
{
    public static class EnemySpawner
    {
        static readonly Random m_Random = new ();
        static float m_nInverseSpawnChance = 90f;

        public static void Update ()
        {
            if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
            {
                Enemy enemy = new (Art.Seeker, Vector2.Zero, Vector2.Zero, 0f);
            }
        }
    }
}
{% endcodeblock %}

接著讓生成機率隨著遊戲時間逐漸增加，每幀減少 `m_nInverseSpawnChance` 的值直到 30。

{% codeblock EnemySpawner.cs lang:csharp %}
public static void Update ()
{
    //...

    if (m_nInverseSpawnChance > 30f)
    {
        m_nInverseSpawnChance -= 0.005f;
    }
}
{% endcodeblock %}

現在敵人都生成在原點，我們希望能讓敵人隨機的出現在畫面上，並且生成時與玩家有些距離。

為了取得玩家位置，必須先對 `Game1` 做些調整，因為 `m_PlayerShip` 不是 static 變數，所以無法像 `Width`, `Height` 一樣存取，解決方法是先宣告一個 static `Game1` 變數 `Instance`，在 constructor 中將 `Instance` 設為 `this`，之後透過 `Game1.Instance` 去存取成員變數。

{% codeblock Game1.cs lang:csharp %}
public class Game1 : Game
{
    public static Game1 Instance { get; private set; }

    //...

    private PlayerShip m_PlayerShip;
    public PlayerShip PlayerShip { get { return m_PlayerShip; } }

    public Game1 ()
    {
        Instance = this;

        //...
    }
}
{% endcodeblock %}

回到 `EnemySpawner`，宣告一個 `GetSpawnPosition` 函式，利用 `Game1.Width` 和 `Game1.Height` 隨機取得一個位置，如果取得的位置和玩家位置距離小於 250 就重新取得。

{% codeblock EnemySpawner.cs lang:csharp %}
public static void Update ()
{
    if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
    {
        //Enemy enemy = new (Art.Seeker, Vector2.Zero, Vector2.Zero, 0f);
        Enemy enemy = new (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f);
    }
}

static Vector2 GetSpawnPosition ()
{
    Vector2 position;

    do
    {
        position = new Vector2 (m_Random.Next (Game1.Width), m_Random.Next (Game1.Height));
    }
    while (Vector2.DistanceSquared (position, Game1.Instance.PlayerShip.Position) < 250 * 250);

    return position;
}
{% endcodeblock %}

現在需要一個容器來儲存生成的 `Enemy`，和 `BulletManager` 一樣，新增一個 EnemyManager.cs，內容和 `BulletManager` 並沒有什麼不同，只是容器的類型由 `Bullet` 換成 `Enemy`。

{% codeblock EnemyManager.cs lang:csharp %}
using Microsoft.Xna.Framework.Graphics;
using System.Collections.Generic;

namespace NeonShooter
{
    public static class EnemyManager
    {
        static readonly List<Enemy> m_EnemyList = [];

        public static void AddBullet (Enemy _enemy)
        {
            m_EnemyList.Add (_enemy);
        }

        public static void Update ()
        {
            foreach (Enemy enemy in m_EnemyList)
            {
                enemy.Update ();
            }

            m_EnemyList.RemoveAll (x => x.IsExpired);
        }

        public static void Draw (SpriteBatch _spriteBatch)
        {
            foreach (Enemy enemy in m_EnemyList)
            {
                enemy.Draw (_spriteBatch);
            }
        }
    }
}
{% endcodeblock %}

{% codeblock EnemyManager.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...

    m_PlayerShip.Update ();
    EnemyManager.Update ();
    BulletManager.Update ();

    //...
}

protected override void Draw (GameTime _gameTime)
{
    //...

    m_SpriteBatch.Begin ();
    m_PlayerShip.Draw (m_SpriteBatch);
    EnemyManager.Draw (m_SpriteBatch);
    BulletManager.Draw (m_SpriteBatch);
    m_SpriteBatch.End ();

    //...
}
{% endcodeblock %}

{% codeblock EnemySpawner.cs lang:csharp %}
public static void Update ()
{
    if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
    {
        //Enemy enemy = new (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f);
        EnemyManager.AddEnemy (new Enemy (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f));
    }
}
{% endcodeblock %}

最後將 `EnemySpawner` 的 `Update` 加到 `Game1` 中，考慮到生成敵人時會用到玩家位置，所以 `Update` 的時機要在 `m_PlayerShip` 後面。

{% codeblock EnemyManager.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...

    m_PlayerShip.Update ();
    EnemySpawner.Update ();
    EnemyManager.Update ();
    BulletManager.Update ();

    //...
}
{% endcodeblock %}

### 2. 碰撞處理
目前生成的敵人還不會動，而且玩家和子彈碰到了也不會有任何作用，我們希望當敵人碰到玩家的時候玩家會死掉，敵人碰到子彈時敵人會死掉，同時子彈也要消失。

判斷碰撞的方法可以很簡單，當兩個物件移動以後發生重疊就可以當作發生了碰撞，範例中將每個物件都給了一個半徑，將物件都視為圓形來處理碰撞，只要檢查兩個物件的距離是否小於半徑之和就代表發生碰撞。

> 當物件移動速度過快時可能會發生該發生碰撞時物件卻穿越的現象，根據要求的精度，這可以是個很複雜且難以處理的問題，這次先不討論。

先將物件都加上半徑。

{% codeblock PlayerShip.cs lang:csharp %}
//...
private Vector2 m_Size = Vector2.Zero;
private float m_Radius = 0f;
private float m_Scale = 1f;

//...
public Vector2 Size { get { return m_Size; } }
public float Radius { get { return m_Radius; } }

public PlayerShip (Texture2D _image, Vector2 _position, float _rotation)
{
    //...
    m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;
}
{% endcodeblock %}

{% codeblock Bullet.cs lang:csharp %}
//...
using System;

//...
private float m_Radius = 0f;
private float m_Scale = 1f;
private bool m_IsExpired = false;

//...
public Vector2 Size { get { return m_Size; } }
public float Radius { get { return m_Radius; } }

public Bullet (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
{
    //...
    m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;
}
{% endcodeblock %}

{% codeblock Enemy.cs lang:csharp %}
//...
using System;

//...
private float m_Radius = 0f;
private float m_Scale = 1f;
private bool m_IsExpired = false;

//...
public Vector2 Size { get { return m_Size; } }
public float Radius { get { return m_Radius; } }
public bool IsExpired { get { return m_IsExpired; } }

public Enemy (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
{
    //...
    m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;
}
{% endcodeblock %}

觀察程式碼可以發現，玩家、子彈、敵人分別儲存在三個 class 中，要將物件資料取出來做碰撞處理並不太方便，這個問題先暫時擱置，等到完成功能以後再回頭來整理，先將 `BulletManager` 和 `EnemyManager` 的容器新增 property 以便存取。

{% codeblock BulletManager.cs lang:csharp %}
static readonly List<Bullet> m_BulletList = [];
public static List<Bullet> BulletList {  get { return m_BulletList; } }
{% endcodeblock %}

{% codeblock EnemyManager.cs lang:csharp %}
static readonly List<Enemy> m_EnemyList = [];
public static List<Enemy> EnemyList {  get { return m_EnemyList; } }
{% endcodeblock %}

在 `Game1` 的中新增 `HandleCollision` 函式，因為子彈會在 `PlayerShip` 呼叫 `Update` 時生成，如果子彈在生成後立即處理和敵人的碰撞，那麼畫面上就只會看到敵人直接消失了而不會看到子彈，我們希望子彈至少能被看見一幀，所以 `HandleCollision` 應該放在 `m_PlayerShip.Update` 之前。

{% codeblock Game1.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...

    HandleCollision ();

    m_PlayerShip.Update ();
    //...
}

void HandleCollision ()
{
    foreach (Enemy enemy in EnemyManager.EnemyList)
    {
        if (!enemy.IsExpired)
        {
            float radius = enemy.Radius + m_PlayerShip.Radius;
            if (Vector2.DistanceSquared (enemy.Position, m_PlayerShip.Position) < radius * radius)
            {
                // TODO: 碰撞處理
            }
        }
    }

    foreach (Enemy enemy in EnemyManager.EnemyList)
    {
        foreach (Bullet bullet in BulletManager.BulletList)
        {
            if (!enemy.IsExpired && !bullet.IsExpired)
            {
                float radius = enemy.Radius + bullet.Radius;
                if (Vector2.DistanceSquared (enemy.Position, bullet.Position) < radius * radius)
                {
                    // TODO: 碰撞處理
                }
            }
        }
    }
}
{% endcodeblock %}

在 `PlayerShip` 中新稱 `Kill` 函式，在玩家被敵人碰撞時呼叫，玩家被碰撞以後將所有物件移除，在 `Game1` 中新增 `Reset` 函式，在玩家被敵人碰撞時呼叫。

{% codeblock PlayerShip.cs lang:csharp %}
public void Kill ()
{
    Game1.Instance.Reset ();
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
void HandleCollision ()
{
    //...
    if (Vector2.DistanceSquared (enemy.Position, m_PlayerShip.Position) < radius * radius)
    {
        m_PlayerShip.Kill ();
    }
    //...
}

public void Reset ()
{
    EnemyManager.EnemyList.Clear ();
    BulletManager.BulletList.Clear ();
}
{% endcodeblock %}

直接呼叫 `Clear` 可以將所有物件從容器中移除，但是會造成程式崩潰，這是因為在 `HandleCollision` 中我們已經用 foreach 在存取容器了，所以應該要改成將所有物件的 `m_IsExpired` 設為 false，在 `Enemy` 和 `Bullet` 中新增 `Kill` 函式，把 `Clear` 改成遍歷容器呼叫 `Kill`。

{% codeblock Enemy.cs lang:csharp %}
public void Kill ()
{
    m_IsExpired = true;
}
{% endcodeblock %}

{% codeblock Bullet.cs lang:csharp %}
public void Kill ()
{
    m_IsExpired = true;
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
public void Reset ()
{
    //EnemyManager.EnemyList.Clear ();
    //BulletManager.BulletList.Clear ();

    foreach (Enemy enemy in EnemyManager.EnemyList)
    {
        enemy.Kill ();
    }

    foreach (Bullet bullet in BulletManager.BulletList)
    {
        bullet.Kill ();
    }
}
{% endcodeblock %}

同樣的，當敵人和子彈碰撞的時候，也呼叫 `Kill`。

{% codeblock Game1.cs lang:csharp %}
void HandleCollision ()
{
    //...
    if (Vector2.DistanceSquared (enemy.Position, bullet.Position) < radius * radius)
    {
        enemy.Kill ();
        bullet.Kill ();
    }
    //...
}
{% endcodeblock %}

最後再將玩家重置到遊戲開始時的位置，在 `PlayerShip` 中新增 `Reset` 函式重新設定玩家的各項屬性。

{% codeblock PlayerShip.cs lang:csharp %}
public void Reset ()
{
    m_Position = new Vector2 (Game1.Width / 2f, Game1.Height / 2f);
    m_Rotation = 0f;
    m_CooldownRemaining = 0;
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
public void Reset ()
{
    m_PlayerShip.Reset ();

    //...
}
{% endcodeblock %}

到目前為止已經完成了敵人生成並且和玩家子彈可以產生互動了，接著要讓敵人開始行動、新增敵人種類，根據不同種類有不同行動模式，但在這之前程式碼已經有點複雜了，下一篇將先對程式碼做一些整理再繼續。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-3.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-3/NeonShooter)

***
參考資料
- *[https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter)*
