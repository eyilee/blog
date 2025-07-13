---
title: MonoGame 範例 NeonShooter 之 7
categories: MonoGame
date: 2025-06-21 14:43:50
updated: 2025-06-21 14:43:50
tags:
---

這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第七篇，本篇將完成以下部分：
1. 加入黑洞
2. 加入粒子

<!-- more -->
### 1. 加入黑洞
黑洞有幾個效果，會根據距離持續地吸引周邊的物體，但是會彈開子彈，自己會不斷的大小縮放，同時也會噴發一些粒子，粒子會在黑洞周圍旋轉直到掉進黑洞裡，首先先建立基本的黑洞 class。

{% codeblock BlackHole.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using System;

namespace NeonShooter
{
    public class BlackHole : Entity
    {
        public BlackHole (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
        {
            m_Image = _image;
            m_Position = _position;
            m_Velocity = _velocity;
            m_Rotation = _rotation;
            m_Size = new Vector2 (_image.Width, _image.Height);
            m_Radius = MathF.Sqrt (m_Size.X * m_Size.X + m_Size.Y * m_Size.Y) / 2f;
        }

        public override void Update ()
        {
        }
    }
}
{% endcodeblock %}

因為要讓黑洞隨時間改變大小，需要覆寫一個新的 `Draw` 函式。

{% codeblock BlackHole.cs lang:csharp %}
//...

public override void Draw (SpriteBatch _spriteBatch)
{
    double time = Game1.Instance.GameTime.TotalGameTime.TotalSeconds;
    m_Scale = 1 + 0.1f * (float)Math.Sin (time * 10);
    _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
}
{% endcodeblock %}

接著要找到黑洞附近的物體，在 `EntityManager` 中加入 `GetNearbyEntities`，這邊使用到 Linq 的 `Where`，根據傳入的 Predicate 函式會回傳一個集合。

{% codeblock EntityManager.cs lang:csharp %}
//...
using System.Collections.Generic;
using System.Linq;

public static class EntityManager
{
    //...

    public static IEnumerable<Entity> GetNearbyEntities (Vector2 _position, float _radius)
    {
        return m_EntityList.Where (x => Vector2.DistanceSquared (_position, x.Position) < _radius * _radius);
    }
}
{% endcodeblock %}

如果物體是子彈就加上一個反向的速度，如果是角色或敵人則根據距離決定增加的速度大小。

{% codeblock Entity.cs lang:csharp %}
//...
public Vector2 Position { get { return m_Position; } }
public Vector2 Velocity { get { return m_Velocity; } set { m_Velocity = value; } }
public float Rotation { get { return m_Rotation; } }
//...
{% endcodeblock %}

{% codeblock EntityManager.cs lang:csharp %}
public override void Update ()
{
    IEnumerable<Entity> entities = EntityManager.GetNearbyEntities (Position, 250f);

    foreach (Entity entity in entities)
    {
        if (entity is Bullet)
        {
            Vector2 velocity = entity.Position - Position;
            velocity.Normalize ();
            velocity *= 0.3f;
            entity.Velocity += velocity;
        }
        else
        {
            Vector2 velocity = Position - entity.Position;
            float distance = velocity.Length ();
            velocity.Normalize ();
            velocity *= float.Lerp (2f, 0f, distance / 250f);
            entity.Velocity += velocity;
        }
    }
}
{% endcodeblock %}

需要注意的是，我們在角色的 `Update` 中直接根據鍵盤輸入給了速度，所以這邊加上去的速度實際是沒有作用的，需要做一點修改，鍵盤輸入的速度也改為增加，在 `Update` 的最後把速度再歸零。

{% codeblock PlayerShip.cs lang:csharp %}
public override void Update ()
{
    //...

    //m_Velocity = direction * 8;
    m_Velocity += direction * 8;
    m_Position += m_Velocity;
    m_Velocity = Vector2.Zero;
    m_Position.X = float.Clamp (m_Position.X, Size.X / 2, (Game1.Width - Size.X / 2));
    m_Position.Y = float.Clamp (m_Position.Y, Size.Y / 2, (Game1.Height - Size.Y / 2));

    //...
}
{% endcodeblock %}

先將 `EnemySpawner` 加入黑洞的生成看看運行的結果，機率為固定數值，因為我們不希望場上有太多黑洞，目前也還沒有加入破壞黑洞的程式碼。

{% codeblock EnemySpawner.cs lang:csharp %}
//...
static float m_nInverseSpawnChance = 90f;
static float m_nInverseBlackHoleChance = 600f;

public static void Update ()
{
    //...

    if (m_Random.Next ((int)m_nInverseBlackHoleChance) == 0)
    {
        EntityManager.AddEntity (new BlackHole (Art.BlackHole, GetSpawnPosition (), Vector2.Zero, 0f));
    }

    if (m_nInverseSpawnChance > 30f)
    {
        m_nInverseSpawnChance -= 0.005f;
    }
}
{% endcodeblock %}

運行以後可以發現有些敵人被困在黑洞中，而且子彈被彈飛以後並沒有轉向，接著我們要解決這問題，在 `EntityManager` 中加入黑洞的碰撞，別忘了角色和子彈也會和黑洞產生碰撞。

{% codeblock EntityManager.cs lang:csharp %}
//...
static readonly List<Enemy> m_EnemyList = [];
static readonly List<BlackHole> m_BlackHoleList = [];

public static void AddEntity (Entity _entity)
{
    //...
    else if (_entity is Enemy)
    {
        m_EnemyList.Add (_entity as Enemy);
    }
    else if (_entity is BlackHole)
    {
        m_BlackHoleList.Add (_entity as BlackHole);
    }
}

public static void Update ()
{
    //...
    m_EnemyList.RemoveAll (x => x.IsExpired);
    m_BlackHoleList.RemoveAll (x => x.IsExpired);
}

public static void HandleCollision ()
{
    //...

    foreach (BlackHole blackHole in m_BlackHoleList)
    {
        if (!blackHole.IsExpired)
        {
            float radius = blackHole.Radius + m_PlayerShip.Radius;
            if (Vector2.DistanceSquared (blackHole.Position, m_PlayerShip.Position) < radius * radius)
            {
                m_PlayerShip.Kill ();
            }
        }

        foreach (Enemy enemy in m_EnemyList)
        {
            if (!enemy.IsExpired && !blackHole.IsExpired)
            {
                float radius = blackHole.Radius + enemy.Radius;
                if (Vector2.DistanceSquared (blackHole.Position, enemy.Position) < radius * radius)
                {
                    enemy.Kill ();
                }
            }
        }

        foreach (Bullet bullet in m_BulletList)
        {
            if (!bullet.IsExpired && !bullet.IsExpired)
            {
                float radius = blackHole.Radius + bullet.Radius;
                if (Vector2.DistanceSquared (blackHole.Position, bullet.Position) < radius * radius)
                {
                    blackHole.Kill ();
                    bullet.Kill ();
                }
            }
        }
    }
}

public static void Reset ()
{
    //...

    foreach (BlackHole blackHole in m_BlackHoleList)
    {
        blackHole.Kill ();
    }
}
{% endcodeblock %}

{% codeblock BlackHole.cs lang:csharp %}
//...

public void Kill ()
{
    m_IsExpired = true;
}
{% endcodeblock %}

在子彈的 `Update` 中計算速度的方向讓它轉向。

{% codeblock Bullet.cs lang:csharp %}
public override void Update ()
{
    m_Position += m_Velocity;

    if (m_Velocity != Vector2.Zero)
    {
        m_Rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
    }

    //...
}
{% endcodeblock %}

### 2. 加入粒子
要讓粒子隨著時間從黑洞旋轉噴出，需要加上一個角度變數，每一幀去改變角度，然後我們希望每幀都能產生一個粒子，但是每次會相隔一段時間。

{% codeblock BlackHole.cs lang:csharp %}
private readonly Random m_Random = new ();

private const int m_CooldownFrames = 15;
private int m_CooldownRemaining = 0;
private bool m_IsSpray = true;

float m_SprayRotation = 0f;

public override void Update ()
{
    //...

    if (m_CooldownRemaining == 0)
    {
        m_CooldownRemaining = m_CooldownFrames;
        m_IsSpray = !m_IsSpray;
    }

    if (m_CooldownRemaining > 0)
    {
        m_CooldownRemaining--;
    }

    m_SprayRotation -= MathF.PI * 2f / 50f;

    if (m_IsSpray)
    {
        float length = 12f + m_Random.NextSingle () * 3f;
        Vector2 velocity = length * new Vector2 ((float)Math.Cos (m_SprayRotation), (float)Math.Sin (m_SprayRotation));
        Color color = ColorUtil.HSVToColor (5f, 0.5f, 0.8f);
        Vector2 offset = Vector2.One * (4f + m_Random.NextSingle () * 4f);
        Vector2 position = Position + 2f * new Vector2 (velocity.Y, -velocity.X) + offset;
        ParticleManager.CreateParticle (Art.LineParticle, position, velocity, color, new Vector2 (1.5f, 1.5f), 190f);
    }
}
{% endcodeblock %}

`m_CooldownFrames` 代表每 15 幀切換一次 `m_IsSpray`，用來判斷當前是否該噴出粒子，`m_SprayRotation` 作為噴出角度每幀會逆時針旋轉 `MathF.PI * 2f / 50f` 的角度，產生粒子的時候根據噴出角度給粒子一個初速度，粒子產生的位置則根據初速度的垂直方向做出位移，這樣粒子就可以看起來像是沿著切線方向噴出，再適當的加上一些隨機位移看起來會更自然。

現在粒子還沒受到黑洞引力影響所以會直接遠離黑洞，我們希望粒子也能受到引力影響圍繞黑洞旋轉，參考萬有引力的公式，在粒子的 `Update` 中根據黑洞的距離平方產生一個速度，並且在粒子靠近黑洞時再增加一個切線方向的速度避免粒子太快掉落黑洞中間。

{% codeblock Particle.cs lang:csharp %}
public void Update ()
{
    //...

    foreach (BlackHole blackHole in EntityManager.BlackHoleList)
    {
        Vector2 direction = blackHole.Position - Position;
        float distance = direction.Length ();
        Vector2 velocity = direction / distance;
        Velocity += 10000 * velocity / (distance * distance + 10000);

        if (distance < 400)
        {
            Velocity += 45 * new Vector2 (velocity.Y, -velocity.X) / (distance + 100);
        }
    }

    if (Math.Abs (Velocity.X) + Math.Abs (Velocity.Y) < 0.00000000001f)
    {
        Velocity = Vector2.Zero;
    }
    else
    {
        Velocity *= 0.96f;
    }
}
{% endcodeblock %}

現在所有粒子都會被黑洞吸引而圍繞黑洞了，下一篇預計將是這系列的最後一篇，我們要在背景加入引力線。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-7.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-7/NeonShooter)
