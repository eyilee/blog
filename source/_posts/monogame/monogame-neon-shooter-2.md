---
title: MonoGame 範例 NeonShooter 之 2
date: 2025-05-01 15:16:08
updated: 2025-05-01 15:16:08
categories: MonoGame
tags:
---

這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第二篇，本篇將完成以下部分：
1. 建立並發射子彈
2. 子彈移動及碰撞

<!-- more -->
### 1. 建立並發射子彈
和建立玩家角色的方式相同，先新增一個 Bullet.cs，宣告一個 class `Bullet`，接著從 class `PlayerShip` 複製出需要的部分包含貼圖、位置、旋轉等，還有對應的 `Draw` 函式。

{% codeblock Bullet.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

namespace NeonShooter
{
    public class Bullet
    {
        private Texture2D m_Image;
        private Vector2 m_Position = Vector2.Zero;
        private Color m_Color = Color.White;
        private float m_Rotation = 0f;
        private Vector2 m_Size = Vector2.Zero;
        private float m_Scale = 1f;

        public Vector2 Position { get { return m_Position; } }
        public float Rotation { get { return m_Rotation; } }
        public Vector2 Size { get { return m_Size; } }

        public Bullet (Texture2D _image, Vector2 _position, float _rotation)
        {
            m_Image = _image;
            m_Position = _position;
            m_Rotation = _rotation;
            m_Size = new Vector2 (_image.Width, _image.Height);
        }

        public void Draw (SpriteBatch _spriteBatch)
        {
            _spriteBatch.Draw (m_Image, m_Position, null, m_Color, m_Rotation, m_Size / 2f, m_Scale, SpriteEffects.None, 0f);
        }
    }
}
{% endcodeblock %}

接著要將子彈以固定的頻率生成在玩家的位置上，範例中在 class `PlayerShip` 中使用了一個變數 `cooldowmRemaining` 來控制何時生成子彈，每一次執行 `Update` 就會減少一次計數，當 `cooldowmRemaining` 等於 0 的時候就代表這次 `Update` 要生成子彈，之後再將 `cooldowmRemaining` 重新設值，達成固定頻率的目的。

我們在 class `PlayerShip` 新增兩個 `int` 變數 `m_CooldownFrames` 和 `m_CooldownRemaining`，`m_CooldownFrames` 代表每幾次 `Update` 要發射一次子彈，會是個固定的值，因此可以宣告為 `const`。

{% codeblock PlayerShip.cs lang:csharp %}
//...
private float m_Scale = 1f;

private const int m_CooldownFrames = 6;
private int m_CooldownRemaining = 0;

public Vector2 Position { get { return m_Position; } }
//...
{% endcodeblock %}

在 `Update` 函式中加入對 `m_CooldownRemaining` 的處理，當 `m_CooldownRemaining` 等於 0 的時候要生成一個 `Bullet` 並且將值重設為 `m_CooldownFrames`，最後無論有無發射子彈每次 `Update` 都要將 `m_CooldownRemaining` 減一。

{% codeblock PlayerShip.cs lang:csharp %}
public void Update ()
{
    //...

    m_Rotation = (float)Math.Atan2 (aimDirection.Y, aimDirection.X);

    if (m_CooldownRemaining == 0)
    {
        Bullet bullet = new Bullet (Art.Bullet, m_Position, m_Rotation);

        m_CooldownRemaining = m_CooldownFrames;
    }

    if (m_CooldownRemaining > 0)
    {
        m_CooldownRemaining--;
    }
}
{% endcodeblock %}

現在我們已經可以不斷的生成子彈了，但是沒有呼叫 `Draw` 函式所以無法顯示在畫面中，而且在 `Update` 中並不適合呼叫 `Draw`，另外，生成的子彈在離開函式以後就會被解構，等於這一段都做了白工。

為了解決這個問題，可以新建一個容器用來儲存生成的 `Bullet`，新增一個 BulletManager.cs，宣告一個 static class `BulletManager`，因為同時會有很多子彈所以可以用 `List` 作為容器，宣告一個 static readonly `List<Bullet>` 變數 `m_BulletList`。

接著新增 `AddBullet` 函式以便我們可以在 class `PlayerShip` 中將生成的子彈加入容器，最後再新增 `Draw` 函式將容器中的所有子彈畫到畫面上。

{% codeblock BulletManager.cs lang:csharp %}
using Microsoft.Xna.Framework.Graphics;
using System.Collections.Generic;

namespace NeonShooter
{
    public static class BulletManager
    {
        static readonly List<Bullet> m_BulletList = [];

        public static void AddBullet (Bullet _bullet)
        {
            m_BulletList.Add (_bullet);
        }

        public static void Draw (SpriteBatch _spriteBatch)
        {
            foreach (Bullet bullet in m_BulletList)
            {
                bullet.Draw (_spriteBatch);
            }
        }
    }
}
{% endcodeblock %}

在 class `PlayerShip` 中呼叫 `AddBullet`，在 class `Game1` 中呼叫 `Draw`。

{% codeblock PlayerShip.cs lang:csharp %}
public void Update ()
{
    //...

    if (m_CooldownRemaining == 0)
    {
        BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position, m_Rotation));

        m_CooldownRemaining = m_CooldownFrames;
    }

    //...
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
protected override void Draw (GameTime _gameTime)
{
    //...

    m_SpriteBatch.Begin ();
    m_PlayerShip.Draw (m_SpriteBatch);
    BulletManager.Draw (m_SpriteBatch);
    m_SpriteBatch.End ();

    //...
}
{% endcodeblock %}

生成子彈的部分已經算是完成了，現在想要一次發射兩顆子彈，而且發射的時候再加上一些偏移，讓兩顆子彈平行。

利用 `Quaternion.CreateFromYawPitchRoll` 可以把弧度轉換成 `Quaternion`，我們是繞 Z 軸旋轉，所以只需要放入 Roll 參數，再呼叫 `Vector2.Transform` 把 `Vector2` 根據傳入的 `Quaternion` 做旋轉，這邊偏移量選擇 35 和 8，子彈會在角色前方距離 35 的地方出現，左右側距離為 8。

{% codeblock PlayerShip.cs lang:csharp %}
public void Update ()
{
    //...

    if (m_CooldownRemaining == 0)
    {
        // BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position, m_Rotation));
        Quaternion aimQuaternion = Quaternion.CreateFromYawPitchRoll (0, 0, m_Rotation);
        BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, -8), aimQuaternion), m_Rotation));
        BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, 8), aimQuaternion), m_Rotation));

        m_CooldownRemaining = m_CooldownFrames;
    }

    //...
}
{% endcodeblock %}

### 2. 子彈移動及碰撞
子彈生成以後會留在原地，而且隨著時間越長，畫面上的子彈會越來越多，所以接著要讓子彈開始移動，並且在離開畫面以後將子彈移除，做法也很簡單，可以想像子彈是撞到了四面牆壁所以消失了，在之後處理子彈碰到敵人時也是相同的概念。

在 class `Bullet` 宣告一個 `Vector2` 變數 `m_Velocity` 代表子彈速度的向量，每次 `Update` 時會根據這個值做位移。

{% codeblock Bullet.cs lang:csharp %}
//...
private Vector2 m_Position = Vector2.Zero;
private Vector2 m_Velocity = Vector2.Zero;
private Color m_Color = Color.White;
//...

public Bullet (Texture2D _image, Vector2 _position, Vector2 _velocity, float _rotation)
{
    m_Image = _image;
    m_Position = _position;
    m_Velocity = _velocity;
    m_Rotation = _rotation;
    m_Size = new Vector2 (_image.Width, _image.Height);
}

public void Update ()
{
    m_Position += m_Velocity;
}
{% endcodeblock %}

回到 class `PlayerShip` 中，已知子彈的朝向是 `m_Rotation`，利用 `Math.Cos` 和 `Math.Sin` 可以得到子彈朝向的單位向量，接著再乘上長度就是需要的速度了。

{% codeblock PlayerShip.cs lang:csharp %}
public void Update ()
{
    //...

    if (m_CooldownRemaining == 0)
    {
        Quaternion aimQuaternion = Quaternion.CreateFromYawPitchRoll (0, 0, m_Rotation);
        Vector2 velocity = 11f * new Vector2 ((float)Math.Cos (m_Rotation), (float)Math.Sin (m_Rotation));
        BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, -8), aimQuaternion), velocity, m_Rotation));
        BulletManager.AddBullet (new Bullet (Art.Bullet, m_Position + Vector2.Transform (new Vector2 (35, 8), aimQuaternion), velocity, m_Rotation));

        m_CooldownRemaining = m_CooldownFrames;
    }

    //...
}
{% endcodeblock %}

在 class `BulletManager` 和 class `Game1` 中加上相關的 `Update`。

{% codeblock BulletManager.cs lang:csharp %}
public static void Update ()
{
    foreach (Bullet bullet in m_BulletList)
    {
        bullet.Update ();
    }
}
{% endcodeblock %}

{% codeblock Game1.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    //...

    m_PlayerShip.Update ();
    BulletManager.Update ();

    //...
}
{% endcodeblock %}

如果要讓子彈超出畫面時消失，要先判斷子彈是否碰撞到畫面的邊緣，畫面的大小已經被記錄在 class `Game1` 的 `Width` 和 `Height` 中了，因為不需要做的太精準，只要判斷子彈的 `m_Position` 是否超出畫面就好了，範例中是使用 `Viewport.Bounds` 的 `Contains` 函式去判斷，如果想要將子彈的體積也納入考慮，`Contains` 另外也有處理 `Rectangle` 的版本，那就需要再計算上 `m_Size` 了，在這裡我們使用簡單的判斷式就可以了。

因為沒辦法在 `Update` 的同時移除掉子彈，宣告一個 `bool` 變數 `m_IsExpired`，當超出畫面以後就設為 `true`，在 class `BulletManager` 中 `Update` 結束時將 `m_IsExpired` 等於 `true` 的子彈從清單中移除。

{% codeblock Bullet.cs lang:csharp %}
//...
private float m_Scale = 1f;
private bool m_IsExpired = false;

//...
public Vector2 Size { get { return m_Size; } }
public bool IsExpired { get { return m_IsExpired; } }

//...

public void Update ()
{
    m_Position += m_Velocity;

    if (m_Position.X < 0 || m_Position.X > Game1.Width || m_Position.Y < 0 || m_Position.Y > Game1.Height)
    {
        m_IsExpired = true;
    }
}
{% endcodeblock %}

{% codeblock BulletManager.cs lang:csharp %}
public static void Update ()
{
    //...
    
    m_BulletList.RemoveAll (x => x.IsExpired);
}
{% endcodeblock %}

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-2.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-2/NeonShooter)

***
參考資料
- *[https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter)*