---
title: MonoGame 範例 AutoPong
date: 2025-02-04 11:17:17
categories: MonoGame
tags:
---
本文將以官方提供的範例 AutoPong 為參考資料，從零開始理解範例中的程式碼，並在內容加入一些些變化。

<!-- more -->
### 設定畫面大小
首先決定畫面的大小，新增兩個成員變數代表寬度和高度，以利後面計算使用。

{% codeblock lang:csharp %}
private int m_Width = 1280;
private int m_Height = 720;
{% endcodeblock %}

接著在 constructor 中設定 GraphicsDeviceManager。

{% codeblock lang:csharp %}
m_Graphics = new GraphicsDeviceManager (this)
{
    PreferredBackBufferWidth = m_Width,
    PreferredBackBufferHeight = m_Height
};

// 如果不是在初始化的時候就設定，還需要再呼叫
// m_Graphics.ApplyChanges (); 
{% endcodeblock %}

目前為止還沒有對畫面移動進行處理，所以場地的大小就由畫面的範圍來決定，超出去看不到的部分這次將不會使用到。

### 製作球和方塊
為了能將物體顯示在畫面上，需要一個 Texture2D 參數，這邊直接建立一個只有 1 pixel 的 Texture2D 物件，也可以使用 Content.Load<Texture2D> 讀取圖片。

{% codeblock lang:csharp %}
public Texture2D m_Texture;

protected override void LoadContent ()
{
    ...

    m_Texture = new Texture2D (m_Graphics.GraphicsDevice, 1, 1);
    m_Texture.SetData ([Color.White]);

    // 使用 Content.Load 讀取圖片
    // m_Texture = Content.Load<Texture2D> ("ball");
}
{% endcodeblock %}

新增三個 Rectangle 成員變數分別代表左右的方塊跟球，因為 MonoGame 本身並沒有包含物理引擎，相關的碰撞檢測必須自己實現，所以這邊都先以方塊來處理，而不是圓形。
預設是以畫面的左上角為原點 (0, 0)，計算置中的時候要記得減去自身的長度。

{% codeblock lang:csharp %}
private Rectangle m_PaddleLeft;
private Rectangle m_PaddleRight;
private Rectangle m_Ball;

protected override void LoadContent ()
{
    ...

    m_PaddleLeft = new Rectangle (0 + 10, m_Height / 2 - 50, 20, 100);
    m_PaddleRight = new Rectangle (m_Width - 30, m_Height / 2 - 50, 20, 100);

    m_Ball = new Rectangle (m_Width / 2 - 5, m_Height / 2 - 5, 10, 10);
}
{% endcodeblock %}

需要注意 Draw 函數的第三個參數，這個 sourceRectangle 指的是在 Texture 上採樣的範圍，如果是 null 的話就會使用整張 Texture，採樣的範圍會轉換成 texel 的方式計算，也就是 uv，
這裡直接使用 m_PaddleLeft 等作為參數，因為 Texture 只有 1 pixel，所以無論如何採樣都會是那個 pixel，如此就能以同個顏色填滿整個範圍。

{% codeblock lang:csharp %}
protected override void Draw (GameTime _gameTime)
{
    GraphicsDevice.Clear (Color.CornflowerBlue);

    m_SpriteBatch.Begin ();

    m_SpriteBatch.Draw (m_Texture, new Vector2 (m_PaddleLeft.X, m_PaddleLeft.Y), m_PaddleLeft, Color.White, 0, Vector2.Zero, 1.0f, SpriteEffects.None, 0.00001f);
    m_SpriteBatch.Draw (m_Texture, new Vector2 (m_PaddleRight.X, m_PaddleRight.Y), m_PaddleRight, Color.White, 0, Vector2.Zero, 1.0f, SpriteEffects.None, 0.00001f);
    m_SpriteBatch.Draw (m_Texture, new Vector2 (m_Ball.X, m_Ball.Y), m_Ball, Color.White, 0, Vector2.Zero, 1.0f, SpriteEffects.None, 0.00001f);

    m_SpriteBatch.End ();

    base.Draw (_gameTime);
}
{% endcodeblock %}

現在應該可以看到畫面上有三個方塊了，接著要讓他們動起來。

### 移動球
要讓球動起來必須要決定方向和速度，因為 Rectangle 的數值型態是 int，用來計算移動可能會產生誤差造成球看起來抖動，所以另外使用 m_BallPosition 來記錄球當前的位置，
m_BallVelocity 代表移動的方向，向量的長度會影響速度的計算，如果要讓前進速度保持一致的話可以呼叫 Normalize。

{% codeblock lang:csharp %}
private Vector2 m_BallPosition;
private Vector2 m_BallVelocity;
private float m_BallSpeed = 15.0f;

protected override void LoadContent ()
{
    ...

    m_BallPosition = new Vector2 (m_Ball.X, m_Ball.Y);
    m_BallVelocity = new Vector2 (1.0f, 0.1f);
}

protected override void Update (GameTime _gameTime)
{
    ...

    m_BallPosition.X += m_BallVelocity.X * m_BallSpeed;
    m_BallPosition.Y += m_BallVelocity.Y * m_BallSpeed;

    base.Update (_gameTime);
}

protected override void Draw (GameTime _gameTime)
{
    ...

    m_Ball.X = (int)m_BallPosition.X;
    m_Ball.Y = (int)m_BallPosition.Y;
    m_SpriteBatch.Draw (m_Texture, new Vector2 (m_Ball.X, m_Ball.Y), m_Ball, Color.White, 0, Vector2.Zero, 1.0f, SpriteEffects.None, 0.00001f);

    ...
}
{% endcodeblock %}

這時候球已經會移動了，但是很快地就跑出畫面外了，現在要加上邊界，讓他碰到邊界以後可以反彈，超出左右邊界時就將 x 軸速度翻轉，超出左右邊界時就將 y 軸速度翻轉，
需要注意球是一個方塊不是一個點，所以要將長度考慮進去。

{% codeblock lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    ...

    m_BallPosition.X += m_BallVelocity.X * m_BallSpeed;
    m_BallPosition.Y += m_BallVelocity.Y * m_BallSpeed;

    if (m_BallPosition.X < 0)
    {
        m_BallPosition.X = 1;
        m_BallVelocity.X *= -1;
    }
    else if (m_BallPosition.X > m_Width - 10)
    {
        m_BallPosition.X = m_Width - 11;
        m_BallVelocity.X *= -1;
    }

    if (m_BallPosition.Y < 0)
    {
        m_BallPosition.Y = 10 + 1;
        m_BallVelocity.Y *= -1;
    }
    else if (m_BallPosition.Y > m_Height - 10)
    {
        m_BallPosition.Y = m_Height - 11;
        m_BallVelocity.Y *= -1;
    }

    ...
}
{% endcodeblock %}

### 移動方塊
在範例中，左邊的方塊是以隨機速度跟隨著球移動的，這邊讓玩家來控制，右邊的方塊則維持跟隨著球。
Keyboard.GetState 可以取得鍵盤的狀態，IsKeyDown 可以檢查某顆按鍵是否正被按住，當上下方向鍵被按住的時候，就讓方塊移動一段距離。

{% codeblock lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    ...

    KeyboardState keyboardState = Keyboard.GetState ();
    if (keyboardState.IsKeyDown (Keys.Up))
    {
        m_PaddleLeft.Y -= 8;
    }
    else if (keyboardState.IsKeyDown (Keys.Down))
    {
        m_PaddleLeft.Y += 8;
    }

    ...
}
{% endcodeblock %}

當右邊的方塊和球 y 軸的距離超過球的長度時，就讓他在 y 軸往球的位置靠近一段距離。

{% codeblock lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    ...

    int paddleCenter = m_PaddleRight.Y + m_PaddleRight.Height / 2;
    if (paddleCenter < m_BallPosition.Y - 10)
    {
        m_PaddleRight.Y -= (int)((paddleCenter - m_BallPosition.Y) * 0.1f);
    }
    else if (paddleCenter > m_BallPosition.Y + 30)
    {
        m_PaddleRight.Y += (int)((m_BallPosition.Y - paddleCenter) * 0.1f);
    }

    ...
}
{% endcodeblock %}

現在方塊有可能會超出邊界，所以再加上檢查。

{% codeblock lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    ...

    LimitPaddle (ref m_PaddleLeft);
    LimitPaddle (ref m_PaddleRight);

    ...
}

private void LimitPaddle (ref Rectangle _paddle)
{
    if (_paddle.Y < 0)
    {
        _paddle.Y = 0;
    }
    else if (_paddle.Y + _paddle.Height > m_Height)
    {
        _paddle.Y = m_Height - _paddle.Height;
    }
}
{% endcodeblock %}

### 球和方塊的碰撞
現在球會直接穿過方塊，我們要讓球碰到方塊的時候可以反彈，就必須檢查兩個物體是否有接觸或重疊。
範例使用 Intersects 函式檢查兩個 Rectangle 是否有相交，忽略了剛好接觸到的情形，因為這並不會有太大影響，所以保持原樣就可以了。

{% codeblock lang:csharp %}
protected override void Update (GameTime _gameTime)
{
    ...

    m_BallPosition.X += m_BallVelocity.X * m_BallSpeed;
    m_BallPosition.Y += m_BallVelocity.Y * m_BallSpeed;

    if (m_PaddleLeft.Intersects (m_Ball))
    {
        m_BallVelocity.X *= -1;
        m_BallPosition.X = m_PaddleLeft.X + m_PaddleLeft.Width;
    }

    if (m_PaddleRight.Intersects (m_Ball))
    {
        m_BallVelocity.X *= -1;
        m_BallPosition.X = m_PaddleRight.X - 10;
    }

    ...
}
{% endcodeblock %}

到目前，我們就完成範例中主要的部份了。

![](/images/monogame-auto-pong.gif)

***
參考資料
- *[https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/AutoPong](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/AutoPong)*