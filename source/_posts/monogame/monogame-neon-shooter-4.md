---
title: MonoGame 範例 NeonShooter 之 4
categories: MonoGame
date: 2025-05-18 21:54:38
updated: 2025-05-18 21:54:38
tags:
---


這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第四篇，本篇將完成以下部分：
1. 整理現有程式碼
2. 新增敵人種類與行動模式

<!-- more -->
### 1. 整理現有程式碼
觀察 `PlayerShip`, `Bullet`, `Enemy` 可以發現有很多變數是相同的，`Draw` 的方法也一樣，可以將相同的部分提取出來成為一個 class `Entity`，遊戲內的所有物件都會繼承 `Entity`。

{% codeblock Entity.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

namespace NeonShooter
{
    public abstract class Entity
    {
        protected Texture2D m_Image;
        protected Vector2 m_Position = Vector2.Zero;
        protected Vector2 m_Velocity = Vector2.Zero;
        protected Color m_Color = Color.White;
        protected float m_Rotation = 0f;
        protected Vector2 m_Size = Vector2.Zero;
        protected float m_Radius = 0f;
        protected float m_Scale = 1f;
        protected bool m_IsExpired = false;

        public Vector2 Position { get { return m_Position; } }
        public float Rotation { get { return m_Rotation; } }
        public Vector2 Size { get { return m_Size; } }
        public float Radius { get { return m_Radius; } }
        public bool IsExpired { get { return m_IsExpired; } }

        public abstract void Update ();

        public virtual void Draw (SpriteBatch _spriteBatch)
        {
            _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
        }
    }
}
{% endcodeblock %}

{% codeblock PlayerShip.cs lang:csharp %}
public class PlayerShip : Entity
{
    //private Texture2D m_Image;
    //private Vector2 m_Position = Vector2.Zero;
    //private Color m_Color = Color.White;
    //private float m_Rotation = 0f;
    //private Vector2 m_Size = Vector2.Zero;
    //private float m_Radius = 0f;
    //private float m_Scale = 1f;

    //...

    //public Vector2 Position { get { return m_Position; } }
    //public float Rotation { get { return m_Rotation; } }
    //public Vector2 Size { get { return m_Size; } }
    //public float Radius { get { return m_Radius; } }

    //...

    public override void Update ()
    {
        //...
    }

    //public void Draw (SpriteBatch _spriteBatch)
    //{
    //    _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
    //}

    //...
}
{% endcodeblock %}

{% codeblock Bullet.cs lang:csharp %}
public class Bullet : Entity
{
    //private Texture2D m_Image;
    //private Vector2 m_Position = Vector2.Zero;
    //private Vector2 m_Velocity = Vector2.Zero;
    //private Color m_Color = Color.White;
    //private float m_Rotation = 0f;
    //private Vector2 m_Size = Vector2.Zero;
    //private float m_Radius = 0f;
    //private float m_Scale = 1f;
    //private bool m_IsExpired = false;

    //public Vector2 Position { get { return m_Position; } }
    //public float Rotation { get { return m_Rotation; } }
    //public Vector2 Size { get { return m_Size; } }
    //public float Radius { get { return m_Radius; } }
    //public bool IsExpired { get { return m_IsExpired; }

    //...

    public override void Update ()
    {
        //...
    }

    //public void Draw (SpriteBatch _spriteBatch)
    //{
    //    _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
    //}

    //...
}
{% endcodeblock %}

{% codeblock Enemy.cs lang:csharp %}
public class Enemy : Entity
{
    //private Texture2D m_Image;
    //private Vector2 m_Position = Vector2.Zero;
    //private Vector2 m_Velocity = Vector2.Zero;
    //private Color m_Color = Color.White;
    //private float m_Rotation = 0f;
    //private Vector2 m_Size = Vector2.Zero;
    //private float m_Radius = 0f;
    //private float m_Scale = 1f;
    //private bool m_IsExpired = false;

    //public Vector2 Position { get { return m_Position; } }
    //public float Rotation { get { return m_Rotation; } }
    //public Vector2 Size { get { return m_Size; } }
    //public float Radius { get { return m_Radius; } }
    //public bool IsExpired { get { return m_IsExpired; } }

    //...

    public override void Update ()
    {
    }

    //public void Draw (SpriteBatch _spriteBatch)
    //{
    //    _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
    //}

    //...
}
{% endcodeblock %}

接著我們也不需要 `BulletManager`, `EnemyManager` 兩個 Manager，可以合併成一個 `EntityManager`，同時 `PlayerShip` 也可作為 `Entity` 一同以 `EntityManager` 來管理。

{% codeblock EntityManager.cs lang:csharp %}
using Microsoft.Xna.Framework.Graphics;
using System.Collections.Generic;

namespace NeonShooter
{
    public static class EntityManager
    {
        static readonly List<Entity> m_EntityList = [];
        static PlayerShip m_PlayerShip;
        static readonly List<Bullet> m_BulletList = [];
        static readonly List<Enemy> m_EnemyList = [];

        public static PlayerShip PlayerShip { get { return m_PlayerShip; } }

        public static void AddEntity (Entity _entity)
        {
            m_EntityList.Add (_entity);

            if (_entity is PlayerShip)
            {
                m_PlayerShip = _entity as PlayerShip;
            }
            else if (_entity is Bullet)
            {
                m_BulletList.Add (_entity as Bullet);
            }
            else if (_entity is Enemy)
            {
                m_EnemyList.Add (_entity as Enemy);
            }
        }

        public static void Update ()
        {
            foreach (Entity entity in m_EntityList)
            {
                entity.Update ();
            }

            m_EntityList.RemoveAll (x => x.IsExpired);
            m_BulletList.RemoveAll (x => x.IsExpired);
            m_EnemyList.RemoveAll (x => x.IsExpired);
        }

        public static void Draw (SpriteBatch _spriteBatch)
        {
            foreach (Entity entity in m_EntityList)
            {
                entity.Draw (_spriteBatch);
            }
        }
    }
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
//private PlayerShip m_PlayerShip;
//public PlayerShip PlayerShip { get { return m_PlayerShip; } }

protected override void LoadContent ()
{
    //...

    //m_PlayerShip = new PlayerShip (Art.Player, new Vector2 (Width / 2f, Height / 2f), 0);
    EntityManager.AddEntity (new PlayerShip (Art.Player, new Vector2 (Width / 2f, Height / 2f), 0));
}

protected override void Update (GameTime _gameTime)
{
    //...

    //m_PlayerShip.Update ();
    EntityManager.Update ();
    EnemySpawner.Update ();
    //EnemyManager.Update ();
    //BulletManager.Update ();

    base.Update (_gameTime);
}

protected override void Draw (GameTime _gameTime)
{
    GraphicsDevice.Clear (Color.CornflowerBlue);

    m_SpriteBatch.Begin ();
    //m_PlayerShip.Draw (m_SpriteBatch);
    //EnemyManager.Draw (m_SpriteBatch);
    //BulletManager.Draw (m_SpriteBatch);
    EntityManager.Draw (m_SpriteBatch);
    m_SpriteBatch.End ();

    base.Draw (_gameTime);
}
{% endcodeblock %}

{% codeblock PlayerShip.cs lang:csharp %}
//BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, -8), aimQuaternion), velocity, m_Rotation));
//BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, 8), aimQuaternion), velocity, m_Rotation));
EntityManager.AddEntity (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, -8), aimQuaternion), velocity, m_Rotation));
EntityManager.AddEntity (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, 8), aimQuaternion), velocity, m_Rotation));
{% endcodeblock %}

{% codeblock EnemySpawner.cs lang:csharp %}
public static void Update ()
{
    if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
    {
        //EnemyManager.AddEnemy (new Enemy (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f));
        EntityManager.AddEntity (new Enemy (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f));
    }

    //...
}

static Vector2 GetSpawnPosition ()
{
    //...
    while (Vector2.DistanceSquared (position, EntityManager.PlayerShip.Position) < 250 * 250);

    return position;
}
{% endcodeblock %}

還有 `HandleCollision`, `Reset` 也都交給 `EntityManager` 處理。

{% codeblock EntityManager.cs lang:csharp %}
public static void Update ()
{
    HandleCollision ();

    //...
}

public static void HandleCollision ()
{
    foreach (Enemy enemy in m_EnemyList)
    {
        if (!enemy.IsExpired)
        {
            float radius = enemy.Radius + m_PlayerShip.Radius;
            if (Vector2.DistanceSquared (enemy.Position, m_PlayerShip.Position) < radius * radius)
            {
                m_PlayerShip.Kill ();
            }
        }
    }

    foreach (Enemy enemy in m_EnemyList)
    {
        foreach (Bullet bullet in m_BulletList)
        {
            if (!enemy.IsExpired && !bullet.IsExpired)
            {
                float radius = enemy.Radius + bullet.Radius;
                if (Vector2.DistanceSquared (enemy.Position, bullet.Position) < radius * radius)
                {
                    enemy.Kill ();
                    bullet.Kill ();
                }
            }
        }
    }
}

public static void Reset ()
{
    m_PlayerShip.Reset ();

    foreach (Enemy enemy in m_EnemyList)
    {
        enemy.Kill ();
    }

    foreach (Bullet bullet in m_BulletList)
    {
        bullet.Kill ();
    }
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...
    //HandleCollision ();

    //...
}

//void HandleCollision ()
//{
//    foreach (Enemy enemy in EnemyManager.EnemyList)
//    {
//        if (!enemy.IsExpired)
//        {
//            float radius = enemy.Radius + m_PlayerShip.Radius;
//            if (Vector2.DistanceSquared (enemy.Position, m_PlayerShip.Position) < radius * radius)
//            {
//                m_PlayerShip.Kill ();
//            }
//        }
//    }

//    foreach (Enemy enemy in EnemyManager.EnemyList)
//    {
//        foreach (Bullet bullet in BulletManager.BulletList)
//        {
//            if (!enemy.IsExpired && !bullet.IsExpired)
//            {
//                float radius = enemy.Radius + bullet.Radius;
//                if (Vector2.DistanceSquared (enemy.Position, bullet.Position) < radius * radius)
//                {
//                    enemy.Kill ();
//                    bullet.Kill ();
//                }
//            }
//        }
//    }
//}

//public void Reset ()
//{
//    m_PlayerShip.Reset ();

//    foreach (Enemy enemy in EnemyManager.EnemyList)
//    {
//        enemy.Kill ();
//    }

//    foreach (Bullet bullet in BulletManager.BulletList)
//    {
//        bullet.Kill ();
//    }
//}
{% endcodeblock %}

{% codeblock PlayerShip.cs lang:csharp %}
public void Kill ()
{
    //Game1.Instance.Reset ();
    EntityManager.Reset ();
}
{% endcodeblock %}

到目前已經將所有物件都移到 `EntityManager` 內處理了，嘗試啟動遊戲後會發現跳出了 exception，原因是在 `EntityManager.Update` 的 foreach 中新增的子彈會存取到正在使用中的 `m_EntityList`，因此需要在 `EntityManager` 新增一個變數，當正在 `Update` 時將新增的 `Entity` 先存在另一個 List 中，等 `Update` 完成後再加入 `m_EntityList`。

{% codeblock PlayerShip.cs lang:csharp %}
static readonly List<Entity> m_EntityList = [];
static readonly List<Entity> m_AddEntityList = [];
//...

static bool m_IsUpdating = false;

//...

public static void AddEntity (Entity _entity)
{
    if (!m_IsUpdating)
    {
        m_EntityList.Add (_entity);
    }
    else
    {
        m_AddEntityList.Add (_entity);
    }

    //...
}

public static void Update ()
{
    m_IsUpdating = true;

    HandleCollision ();

    foreach (Entity entity in m_EntityList)
    {
        entity.Update ();
    }

    m_IsUpdating = false;

    foreach (Entity entity in m_AddEntityList)
    {
        AddEntity (entity);
    }
    m_AddEntityList.Clear ();

    //...
}
{% endcodeblock %}

如此一來，遊戲中的物件都已經改由 `EntityManager` 來管理，並且將重複的部分提取到 `Entity` 當中節省了許多程式碼，接下來繼續完成敵人的設計。

### 2. 新增敵人種類與行動模式
接下來開始幫敵人加上行動，範例提供了兩種敵人的貼圖，我們希望敵人可以有兩種行動模式，一種是追隨玩家，一種則是會隨機遊蕩。

先從實作追隨玩家開始，計算玩家和敵人之間的向量，轉換成單位向量以後乘以長度作為移動向量，並且配合移動向量改變角度。

{% codeblock Enemy.cs lang:csharp %}
public override void Update ()
{
    m_Velocity = EntityManager.PlayerShip.Position - m_Position;
    m_Velocity.Normalize ();
    m_Velocity *= 5.0f;

    m_Position += m_Velocity;

    if (m_Velocity != Vector2.Zero)
    {
        m_Rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
    }
}
{% endcodeblock %}

運行遊戲試玩一下以後會發現，敵人轉彎的太即時了，始終都會以直線距離追趕玩家，體驗不是很好，範例提供的方法是每一幀都會參考原本的速度再加上新增的速度，移動過後再將速度縮小一些不讓速度無限膨脹，這樣敵人在移動時就能保有慣性，看起來會比較合理一些。

{% codeblock Enemy.cs lang:csharp %}
public override void Update ()
{
    //m_Velocity = EntityManager.PlayerShip.Position - m_Position;
    //m_Velocity.Normalize ();
    //m_Velocity *= 5.0f;
    Vector2 vector = EntityManager.PlayerShip.Position - m_Position;
    vector.Normalize ();
    vector *= 0.9f;

    m_Velocity += vector;
    m_Position += m_Velocity;

    if (m_Velocity != Vector2.Zero)
    {
        m_Rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
    }

    m_Velocity *= 0.8f;
}
{% endcodeblock %}

接著實作隨機的遊蕩，但 `Update` 中已經有了一種追隨玩家的程式碼，需要有方法將兩種行動模式區分開來。

一種方法是新增兩個 class 繼承 `Enemy` 分別代表兩種敵人，如此 `Update` 就可以有獨立的邏輯，但問題是每當新增一種敵人就會增加一種 class，我們的 `Enemy` 實際上只有移動模式上的不同，如果每新增一種行動模式就增加一個 class 會顯得太冗餘。

另一種方法是增加一個變數用來判斷要執行哪個行動模式，如此並不需要新增 class，我認為這個方法並無太大問題，但是要多維護一個變數和類型的對照表也不是很方便。

範例提供的方法是利用迭代器 `IEnumerable` 的特性，將行動模式包裝成迭代器在 `Update` 時呼叫，而且因為迭代器會保存狀態，當有需要時可以不必增加成員變數只使用區域變數就可以了。

新增一個 `IEnumerator<int>` 變數 `m_Behaviour`，將追趕玩家的程式碼包裝成迭代器 `ChasePlayer`，因為不希望敵人主動停下來，使用無窮迴圈讓迭代器可以一直執行下去。

{% codeblock Enemy.cs lang:csharp %}
public class Enemy : Entity
{
    IEnumerator<int> m_Behaviour;

    public Enemy (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
    {
        ...

        m_Behaviour = ChasePlayer ();
    }

    public override void Update ()
    {
        //Vector2 vector = EntityManager.PlayerShip.Position - m_Position;
        //vector.Normalize ();
        //vector *= 0.9f;

        //m_Velocity += vector;
        if (m_Behaviour != null)
        {
            if (!m_Behaviour.MoveNext ())
            {
                m_Behaviour = null;
            }
        }

        m_Position += m_Velocity;

        //if (m_Velocity != Vector2.Zero)
        //{
        //    m_Rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
        //}

        m_Velocity *= 0.8f;
    }
    
    IEnumerator<int> ChasePlayer ()
    {
        while (true)
        {
            Vector2 vector = EntityManager.PlayerShip.Position - m_Position;
            vector.Normalize ();
            vector *= 0.9f;

            m_Velocity += vector;

            if (m_Velocity != Vector2.Zero)
            {
                m_Rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
            }

            yield return 0;
        }
    }

    ...
}
{% endcodeblock %}

現在可以開始實作隨機遊蕩模式 `RandomMove`，這個敵人我們希望他可以隨機的朝某一個方向前進，直到到達地圖邊界以後再重新選擇方向，完成以後先將 `m_Behaviour` 改為 `RandomMove` 測試看看。

{% codeblock Enemy.cs lang:csharp %}
static readonly Random m_Random = new ();

IEnumerator<int> m_Behaviour;

public Enemy (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
{
    m_Image = _image;
    m_Position = _position;
    m_Velocity = _velocity;
    m_Rotation = _rotation;
    m_Size = new Vector2 (_image.Width, _image.Height);
    m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;

    //m_Behaviour = ChasePlayer ();
    m_Behaviour = RandomMove ();
}

IEnumerator<int> RandomMove ()
{
    float direction = m_Random.NextSingle () * MathF.PI * 2;

    while (true)
    {
        Vector2 vector = new (MathF.Cos (direction), MathF.Sin (direction));
        vector *= 0.4f;

        m_Velocity += vector;

        m_Rotation += 0.05f;

        if (m_Position.X < 0 || m_Position.X > Game1.Width || m_Position.Y < 0 || m_Position.Y > Game1.Height)
        {
            direction = m_Random.NextSingle () * MathF.PI * 2;
        }

        yield return 0;
    }
}
{% endcodeblock %}

這時候發現有些敵人會在地圖的邊界來回移動無法離開，這是因為每一幀都做了邊界檢查，下一幀還來不及離開邊界，可以加個 for 迴圈每三幀才檢查一次，另外也可以限制回彈的角度，以地圖中心為基準的 90 度扇形角度。

{% codeblock Enemy.cs lang:csharp %}
if (m_Position.X < 0 || m_Position.X > Game1.Width || m_Position.Y < 0 || m_Position.Y > Game1.Height)
{
    //direction = m_Random.NextSingle () * MathF.PI * 2;
    Vector2 toCenter = new (Game1.Width / 2f - m_Position.X, Game1.Height / 2f - m_Position.Y);
    direction = MathF.Atan2 (toCenter.Y, toCenter.X) + (m_Random.NextSingle () - 0.5f) * MathF.PI / 2f;
}
{% endcodeblock %}

新增 `CreateSeeker`, `CreateWanderer` 函式用來生成兩種不同的敵人，將 `m_Behaviour` 的設定從 constructor 移到生成函式中處理。

{% codeblock Enemy.cs lang:csharp %}
public Enemy (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
{
    m_Image = _image;
    m_Position = _position;
    m_Velocity = _velocity;
    m_Rotation = _rotation;
    m_Size = new Vector2 (_image.Width, _image.Height);
    m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;

    //m_Behaviour = RandomMove ();
}

public static Enemy CreateSeeker (Vector2 _position, Vector2 _velocity, float _rotation)
{
    Enemy enemy = new (Art.Seeker, _position, _velocity, _rotation);
    enemy.m_Behaviour = enemy.ChasePlayer ();
    return enemy;
}

public static Enemy CreateWanderer (Vector2 _position, Vector2 _velocity, float _rotation)
{
    Enemy enemy = new (Art.Wanderer, _position, _velocity, _rotation);
    enemy.m_Behaviour = enemy.RandomMove ();
    return enemy;
}
{% endcodeblock %}

最後在 `EnemySpawner` 中修改生成敵人的方法。

{% codeblock EnemySpawner.cs lang:csharp %}
public static void Update ()
{
    if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
    {
        //EntityManager.AddEntity (new Enemy (Art.Seeker, GetSpawnPosition (), Vector2.Zero, 0f));
        EntityManager.AddEntity (Enemy.CreateSeeker (GetSpawnPosition (), Vector2.Zero, 0f));
    }

    if (m_Random.Next ((int)m_nInverseSpawnChance) == 0)
    {
        EntityManager.AddEntity (Enemy.CreateWanderer (GetSpawnPosition (), Vector2.Zero, 0f));
    }

    ...
}
{% endcodeblock %}

這篇我們完成了將物件重複的部分提取成 `Entity` 減少了很多程式碼，利用迭代器生成了兩種不同行動模式的敵人，下一篇開始預計將開始加入特效。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-4.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-4/NeonShooter)

***
參考資料
- *[https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter)*
