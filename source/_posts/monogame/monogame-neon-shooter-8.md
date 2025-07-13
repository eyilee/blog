---
title: MonoGame 範例 NeonShooter 之 8
categories: MonoGame
date: 2025-07-13 23:39:35
updated: 2025-07-13 23:39:35
tags:
---


這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第八篇，本篇將完成以下部分：
1. 加入引力線
2. 施加引力

<!-- more -->
### 1. 加入引力線
想像遊戲場地是一張網子，當中間產生引力時就像網子往螢幕內的方向塌陷下去，要製作出一張網子，我們先定義一系列的點，這些點互相連接形成網子，新增 `Grid`。

每個點包含他的位置，雖然畫面是 2D 的，但是因為我們希望能模擬塌陷的感覺，所以使用 Vector3 來記錄位置，渲染畫面的時候再投影回平面，平均的分布點在畫面上，將網子先建立起來。

{% codeblock Grid.cs lang:csharp %}
using Microsoft.Xna.Framework;

namespace NeonShooter
{
    public class Grid
    {
        class Point
        {
            public Vector3 Position;

            public Point (Vector3 _position)
            {
                Position = _position;
            }
        }

        readonly Point[,] m_Points;

        public Grid (int _spacing)
        {
            int columns = Game1.Width / _spacing + 1;
            int rows = Game1.Height / _spacing + 1;
            m_Points = new Point[columns + 1, rows + 1];

            for (int x = 0; x < columns; x++)
            {
                for (int y = 0; y < rows; y++)
                {
                    m_Points[x, y] = new Point (new Vector3 (x * _spacing, y * _spacing, 0));
                }
            }
        }
    }
}
{% endcodeblock %}

接著要模擬引力對點作用，我們希望彼此相連的點能互相拉扯，但在失去力的作用以後要回復原來的樣子，可以看做點之間有一條線連接著，如果點受到力的作用產生位移了，可以透過線將力傳遞給相連的點。

因此我們可以定義一條線由兩個點組成，然後一一產生這些線，目前這些線還沒有處理到力的部分，先暫時跳過。

{% codeblock Grid.cs lang:csharp %}
//...
using System.Collections.Generic;

namespace NeonShooter
{
    public class Grid
    {
        //...

        class Spring
        {
            public Point End1;
            public Point End2;

            public Spring (Point _end1, Point _end2)
            {
                End1 = _end1;
                End2 = _end2;
            }
        }

        readonly Point[,] m_Points;
        readonly Spring[] m_Springs;

        public Grid (int _spacing)
        {
            //...

            List<Spring> springs = [];
            for (int x = 0; x < columns; x++)
            {
                for (int y = 0; y < rows; y++)
                {
                    if (x > 0)
                    {
                        springs.Add (new Spring (m_Points[x - 1, y], m_Points[x, y]));
                    }

                    if (y > 0)
                    {
                        springs.Add (new Spring (m_Points[x, y - 1], m_Points[x, y]));
                    }
                }
            }

            m_Springs = springs.ToArray ();
        }
    }
}
{% endcodeblock %}

新增 `Draw` 函式，由於我們的點是有深度的，需要模擬透視的效果，從視線到平面可以視為一個四角錐，設定我們的視線與平面的距離為 2000，深度 z 的點與視線的距離就是 z + 2000，透過 `ToVec2` 簡單的比例計算將點投影到平面上。

{% codeblock Grid.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using System;
using System.Collections.Generic;

public void Draw (SpriteBatch _spriteBatch)
{
    int columns = m_Points.GetLength (0);
    int rows = m_Points.GetLength (1);
    Color color = new (30, 30, 139, 85);

    for (int x = 0; x < columns; x++)
    {
        for (int y = 0; y < rows; y++)
        {
            if (x > 0)
            {
                DrawLine (_spriteBatch, ToVec2 (m_Points[x - 1, y].Position), ToVec2 (m_Points[x, y].Position), color);
            }

            if (y > 0)
            {
                DrawLine (_spriteBatch, ToVec2 (m_Points[x, y - 1].Position), ToVec2 (m_Points[x, y].Position), color);
            }
        }
    }
}

public static Vector2 ToVec2 (Vector3 _vector3)
{
    float factor = (_vector3.Z + 2000) / 2000;
    return (new Vector2 (_vector3.X - Game1.Width / 2f, _vector3.Y - Game1.Height / 2f)) * factor + new Vector2 (Game1.Width / 2f, Game1.Height / 2f);
}

public static void DrawLine (SpriteBatch _spriteBatch, Vector2 _start, Vector2 _end, Color _color, float _thickness = 2f)
{
    Vector2 delta = _end - _start;
    float rotation = MathF.Atan2 (delta.Y, delta.X);
    _spriteBatch.Draw (Art.Pixel, _start, null, _color, rotation, new Vector2 (0, 0.5f), new Vector2 (delta.Length (), _thickness), SpriteEffects.None, 0f);
}
{% endcodeblock %}

將 `Grid` 設定好，運行遊戲後確認線是否有正確顯示。

{% codeblock Game1.cs lang:csharp %}
public GameTime GameTime { get; private set; }
public Grid Grid { get; private set; }

protected override void Initialize ()
{
    Grid = new Grid (80);

    base.Initialize ();
}

protected override void Draw (GameTime _gameTime)
{
    //...

    m_SpriteBatch.Begin (SpriteSortMode.Deferred, BlendState.Additive);
    EntityManager.Draw (m_SpriteBatch);
    ParticleManager.Draw (m_SpriteBatch);
    Grid.Draw (m_SpriteBatch);
    m_SpriteBatch.End ();

    //...
}
{% endcodeblock %}

### 2. 施加引力
生成物體的時候，我們希望施加引力給周圍的點，當點受到引力的作用時會產生加速度，進而發生位移，因為我們希望可以透過線拉扯連接的點，這時候線的長度會發生變化，如果要保持線的長度那就得施加一個力給連接的點，這個力的計算可以參考彈力公式，線的長度變化越大力就越大。

新增點的加速度，線的原始長度與彈力係數，為了不讓力持續累積作用，還要增加一個衰減係數。

點的部分，先不考慮質量，以牛頓第二運動定律 F=MA 簡化來看，可以把力直接視為加速度。

{% codeblock Grid.cs lang:csharp %}
class Point
{
    public Vector3 Position;
    public Vector3 Velocity;
    public Vector3 Acceleration;
    public float Damping;

    public Point (Vector3 _position)
    {
        Position = _position;
        Damping = 0.98f;
    }

    public void ApplyForce (Vector3 _force)
    {
        Acceleration += _force;
    }

    public void Update ()
    {
        Velocity += Acceleration;
        Position += Velocity;

        Acceleration = Vector3.Zero;

        if (Velocity.LengthSquared () < 0.001f * 0.001f)
        {
            Velocity = Vector3.Zero;
        }

        Velocity *= Damping;
    }
}
{% endcodeblock %}

線的部分，當兩點距離變長時才會計算彈力，距離變短時的推力則忽略，我們不需要這個效果，需要注意的是若兩個點有相對速度，可以視為有一個額外的力施加在線上，兩個點受到的力方向相反。

{% codeblock Grid.cs lang:csharp %}
class Spring
{
    public Point End1;
    public Point End2;
    public float TargetLength;
    public float SpringRate;
    public float Damping;

    public Spring (Point _end1, Point _end2)
    {
        End1 = _end1;
        End2 = _end2;
        TargetLength = Vector3.Distance (End1.Position, End2.Position);
        SpringRate = 0.1f;
        Damping = 0.1f;
    }

    public void Update ()
    {
        Vector3 vector = End1.Position - End2.Position;

        float length = vector.Length ();
        if (length <= TargetLength)
        {
            return;
        }

        vector = (vector / length) * (length - TargetLength);
        Vector3 diffVelocity = End1.Velocity - End2.Velocity;
        Vector3 force = SpringRate * vector + diffVelocity * Damping;

        End1.ApplyForce (-force);
        End2.ApplyForce (force);
    }
}
{% endcodeblock %}

新增 `ApplyForce` 函式，根據距離線性地施加引力在點上，範例中有三種引力，，這裡就只做一種，

{% codeblock Grid.cs lang:csharp %}
public void ApplyForce (float _force, Vector2 _position, float _radius)
{
    ApplyForce (_force, new Vector3 (_position, 0), _radius);
}

public void ApplyForce (float _force, Vector3 _position, float _radius)
{
    foreach (Point point in m_Points)
    {
        float dist2 = Vector3.DistanceSquared (_position, point.Position);
        if (dist2 < _radius * _radius)
        {
            point.ApplyForce (10 * _force * (_position - point.Position) / (100 + dist2));
        }
    }
}
{% endcodeblock %}

新增 `Update` 函式，注意 `Spring` 的 `Update` 要早於 `Point`，點在移動前要先施加線的拉力。

{% codeblock Grid.cs lang:csharp %}
public void Update ()
{
    foreach (Spring spring in m_Springs)
    {
        spring.Update ();
    }

    foreach (Point point in m_Points)
    {
        point.Update ();
    }
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...

    EntityManager.Update ();
    EnemySpawner.Update ();
    ParticleManager.Update ();
    Grid.Update ();

    base.Update (_gameTime);
}
{% endcodeblock %}

接著讓玩家的位置產生引力，看看效果如何。

{% codeblock PlayerShip.cs lang:csharp %}
public override void Update ()
{
    //...

    Game1.Instance.Grid.ApplyForce (10, new Vector3 (Position, 0), 50);
}
{% endcodeblock %}

可以發現點不斷的被玩家吸引，最後都貼著擠成一團了，我們希望這些點移動以後要能回到原位。

做法是將邊緣的點複製一遍，與原本的點用一條線相連，而複製出來的點必須不受到力的影響始終保持在原地。

為了讓點不受到力的影響，在 `Point` 中新增一個變數 `Mass` 用來表示引力作用大小的參數。

{% codeblock Grid.cs lang:csharp %}
class Point
{
    //...
    public float Damping;
    public float Mass;

    public Point (Vector3 _position, float _mass)
    {
        //...
        Mass = _mass;
    }

    public void ApplyForce (Vector3 _force)
    {
        Acceleration += _force * Mass;
    }
}
{% endcodeblock %}

接著修改初始化時新增的點，原有的點 `Mass` 設為 1，固定的點 `Mass` 設為 0。

{% codeblock Grid.cs lang:csharp %}
public Grid (int _spacing)
{
    //...

    Point[,] fixPoints = new Point[columns, rows];
    for (int x = 0; x < columns; x++)
    {
        for (int y = 0; y < rows; y++)
        {
            m_Points[x, y] = new Point (new Vector3 (x * _spacing, y * _spacing, 0), 1);
            fixPoints[x, y] = new Point (new Vector3 (x * _spacing, y * _spacing, 0), 0);
        }
    }

    List<Spring> springs = [];
    for (int x = 0; x < columns; x++)
    {
        for (int y = 0; y < rows; y++)
        {
            if (x == 0 || x == columns - 1 || y == 0 || y == rows - 1)
            {
                springs.Add (new Spring (m_Points[x, y], fixPoints[x, y]));
            }

            //...
        }
    }

    m_Springs = springs.ToArray ();
}
{% endcodeblock %}

如此一來引力線就算是完成了，另外兩種引力計算的方式因為並不是學習重點所以就跳過了，這個系列到本篇為止可以算是完成了，範例中有許多細節在這裡是沒有完成的，程式碼的部分也較為潦草，有時間的話會再修改一些細節進行整理發佈在 Github 中，最後來看看成果。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-8.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-8/NeonShooter)
