---
title: MonoGame 範例 NeonShooter 之 5
categories: MonoGame
date: 2025-06-01 23:23:04
updated: 2025-06-01 23:23:04
tags:
---


這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第五篇，本篇將完成以下部分：
1. 理解 BloomComponent
2. 加入 Bloom 特效

<!-- more -->
### 1. 理解 BloomComponent
範例中使用的 Bloom 特效取自較早的 XNA 教學，連結已經失效了，相關的程式碼雖然還能在 [Github](https://github.com/SimonDarksideJ/XNAGameStudio/wiki/Bloom-Postprocess) 上找到，但是教學過程已經遺失了，我們只能從程式碼中自行研究了。

在開始使用之前必須先來了解程式碼的內容，Bloom 相關的程式碼總共兩個檔案 BloomComponent.cs 和 BloomSettings.cs，BloomComponent.cs 是實作特效主要的檔案，BloomSettings.cs 負責調整一些參數，另外還有 BloomCombine.fx, BloomExtract.fx, GaussianBlur.fx 三個 Shader 在最一開始已經加入 Content Pipeline 了。

{% codeblock BloomComponent.cs lang:csharp %}
public class BloomComponent : DrawableGameComponent
{% endcodeblock %}

首先要來看 `BloomComponent`，繼承了 `DrawableGameComponent`，文件中的說明是這樣的：

> A drawable object that, when added to the Game.Components collection of a Game instance, will have it's Draw(GameTime) method called when Game.Draw(GameTime) is called.

意思就是只要將 `BloomComponent` 加入 `Components` 以後，當 `base.Draw` 被呼叫時就會呼叫 `BloomComponent.Draw`，那如果有兩個 `DrawableGameComponent` 呢？

為了避免理解錯誤，接著去找了 MonoGame 的原始碼來看，在 `Game.Draw` 中只有遍歷了 `_drawables` 變數呼叫 `Draw`，這個 `_drawables` 變數是一個 `IDrawable` 的容器，而 `IDrawable` 只有用在 `DrawableGameComponent` 上，也就是說 `Game.Draw` 中只會遍歷所有 `Components` 中的 `DrawableGameComponent`，順序則是根據 `DrawableGameComponent` 的 `DrawOrder` 變數。

{% codeblock BloomComponent.cs lang:csharp %}
Effect bloomExtractEffect;
Effect bloomCombineEffect;
Effect gaussianBlurEffect;

RenderTarget2D sceneRenderTarget;
RenderTarget2D renderTarget1;
RenderTarget2D renderTarget2;
{% endcodeblock %}

接著往下看，可以看到有三個 `Effect` 變數，三個 `RenderTarget2D` 變數， `Effect` 主要是用來處理渲染過程中的效果，如變形、光照、貼圖、光暈等視覺效果，至少會包含一個 Pixel Shader 和一個 Vertex Shader，`RenderTarget2D` 是用來儲存渲染後的結果，過程中有多次渲染所以需要多個來儲存，同時還繼承了 `Texture2D` 因為包含了一個 2D 貼圖資料，可以做為 `Texture2D` 傳入任何需要的函式。 

再往下可以發現兩個函式 `LoadContent` 和 `UnloadContent` 並沒有在程式碼內被呼叫過，這是因為在 `DrawableGameComponent` 中的 `Initialize`, `Dispose` 分別會呼叫到，`Initialize` 會在兩個時機被呼叫，第一是當 `Game.Initialize` 之前已將 `DrawableGameComponent` 加入 `Components`，則會在 `Game.Initialize` 中被呼叫，第二是在 `Game.Initialize` 之後加入 `Components` 時呼叫。

到目前為止 `BloomComponent` 主要架構上已經釐清了，剩下是內容的細節。

{% codeblock BloomComponent.cs lang:csharp %}
bloomExtractEffect = this.Game.Content.Load<Effect>("Shaders/BloomExtract");
bloomCombineEffect = this.Game.Content.Load<Effect>("Shaders/BloomCombine");
gaussianBlurEffect = this.Game.Content.Load<Effect>("Shaders/GaussianBlur");
{% endcodeblock %}

先從 `LoadContent` 開始，在第一篇中我們直接拿了範例的 mgcb 檔，Shader 檔在 mgcb Build 過後已經轉成了可以被 `Content.Load` 讀取的格式，具體的過程可以在 MonoGame 原始碼的 EffectProcessor.cs 中看到，在之後只要將 `Effect` 作為參數傳入 `SpriteBatch` 就可以了。

{% codeblock BloomComponent.cs lang:csharp %}
// Look up the resolution and format of our main backbuffer.
PresentationParameters pp = GraphicsDevice.PresentationParameters;

int width = pp.BackBufferWidth;
int height = pp.BackBufferHeight;

SurfaceFormat format = pp.BackBufferFormat;

// Create a texture for rendering the main scene, prior to applying bloom.
sceneRenderTarget = new RenderTarget2D(GraphicsDevice, width, height, false,
                                       format, pp.DepthStencilFormat, pp.MultiSampleCount,
                                       RenderTargetUsage.DiscardContents);
{% endcodeblock %}

創建 `RenderTarget2D` 時需要將 `GraphicsDevice` 作為參數傳入，這是因為 `RenderTarget2D` 會使用到顯示卡的記憶體，因此如果呼叫了 `GraphicsDevice.Reset` 那麼 `RenderTarget2D` 也必須重新創建，後面的參數也都使用與 `GraphicsDevice` 當前的設定相同，最後一個參數 `RenderTargetUsage.DiscardContents` 則是說當畫面在渲染時，切換 RenderTarget 之後資料不須保留，因為這裡渲染的結果將作為貼圖來使用，所以不需要保留。

{% codeblock BloomComponent.cs lang:csharp %}
// Create two rendertargets for the bloom processing. These are half the
// size of the backbuffer, in order to minimize fillrate costs. Reducing
// the resolution in this way doesn't hurt quality, because we are going
// to be blurring the bloom images in any case.
width /= 2;
height /= 2;

renderTarget1 = new RenderTarget2D(GraphicsDevice, width, height, false, format, DepthFormat.None);
renderTarget2 = new RenderTarget2D(GraphicsDevice, width, height, false, format, DepthFormat.None);
{% endcodeblock %}

接著的兩個 `RenderTarget2D` 在寬高只有畫面的一半，將作為模糊效果使用，即使減少畫面的精度也不影響表現，同時也節省一些開銷。

{% codeblock BloomComponent.cs lang:csharp %}
public void BeginDraw()
{
    if (Visible)
    {
        GraphicsDevice.SetRenderTarget(sceneRenderTarget);
    }
}
{% endcodeblock %}

在 `BeginDraw` 內只有做一件事，把 `GraphicsDevice` 的 RenderTarget 換成 `sceneRenderTarget`，`NeonShooterGame.Draw` 的最前面就呼叫了 `BeginDraw`，這樣直到呼叫 `Draw` 之前 `spriteBatch.Draw` 的貼圖就都會被渲染到 `sceneRenderTarget` 上。

最後來看 `Draw` 函式，`BloomComponent` 中所有的效果都在這個函式內實現。

{% codeblock BloomComponent.cs lang:csharp %}
GraphicsDevice.SamplerStates[1] = SamplerState.LinearClamp;
...
GraphicsDevice.Textures[1] = sceneRenderTarget;
{% endcodeblock %}

第一行的 `SamplerState` 用來描述如何對貼圖採樣，`SamplerStates[1]` 則代表是對應 s1 Sampler，而在函式的最後還設定了 `Textures[1]` 為 `sceneRenderTarget`，對應的則是 t1 Texture。

實際上運行以後，按 B 開啟 Bloom 效果會發現，畫面變得十分模糊，經過追查以後發現 `Textures[1]` 並沒有在 Shader 中作用，目前在 Github 上也有提出了 [Issue](https://github.com/MonoGame/MonoGame.Samples/issues/83)，而根據官方人員在先前的文章[回覆的內容](https://gist.github.com/Jjagg/096c5218d21c5e2944a88dbc1b6947b3)上述的用法應該是無誤的，所以只能暫時認為是未修復的 BUG，解法也很簡單，將寫法改成舊式的寫法即可，在 BloomCombine.fx 中手動定義 `BaseSampler`，將 `sceneRenderTarget` 以參數傳入 shader。

{% codeblock BloomCombine.fx lang:hlsl %}
//sampler BaseSampler : register(s1);
sampler BaseSampler : register(s1)
{
    Texture = (BaseTexture);
    Filter = Linear;
    AddressU = clamp;
    AddressV = clamp;
};
{% endcodeblock %}

{% codeblock BloomComponent.cs lang:csharp %}
//GraphicsDevice.Textures[1] = sceneRenderTarget;
bloomCombineEffect.Parameters["BaseTexture"].SetValue(sceneRenderTarget);
{% endcodeblock %}

加上 Bloom 的效果分為以下四個步驟，在程式碼中分別是 Pass 1~4：
1. 將 `sceneRenderTarget` 使用 `bloomExtractEffect` 取出亮的部分渲染到 `renderTarget1` 上。
2. 將 `renderTarget1` 使用 `gaussianBlurEffect` 加上水平方向的高斯模糊渲染到 `renderTarget2` 上。
3. 將 `renderTarget2` 使用 `gaussianBlurEffect` 加上垂直方向的高斯模糊渲染到 `renderTarget1` 上。
4. 將 `renderTarget1` 和 `sceneRenderTarget` 使用 `bloomCombineEffect` 渲染到預設的 RenderTarget。

Shader 和高斯模糊的具體內容在這個範例裡就先不深究了，以了解使用方式為主。

### 2. 加入 Bloom 特效
了解 BloomComponent 以後就可以把檔案直接複製過來使用，不再另外更改。

{% codeblock Game1.cs lang:csharp %}
private GraphicsDeviceManager m_Graphics;
private SpriteBatch m_SpriteBatch;
private BloomComponent m_BloomComponent;

...

public Game1 ()
{
    ...

    m_BloomComponent = new BloomComponent (this);
    Components.Add (m_BloomComponent);
    m_BloomComponent.Settings = new BloomSettings (null, 0.25f, 4, 2, 1, 1.5f, 1);
    m_BloomComponent.Visible = true;

    Content.RootDirectory = "Content";
    IsMouseVisible = true;
}

...

protected override void Draw (GameTime _gameTime)
{
    m_BloomComponent.BeginDraw ();

    GraphicsDevice.Clear (Color.CornflowerBlue);

    ...

    base.Draw (_gameTime);
}
{% endcodeblock %}

下一篇預計將加入粒子效果。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-5.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-5/NeonShooter)

***
參考資料
- *[Bloom-Postprocess](https://github.com/SimonDarksideJ/XNAGameStudio/wiki/Bloom-Postprocess)*
- *[Class DrawableGameComponent](https://docs.monogame.net/api/Microsoft.Xna.Framework.DrawableGameComponent.html)*
- *[Class Effect](https://docs.monogame.net/api/Microsoft.Xna.Framework.Graphics.Effect.html)*
- *[What Is a Configurable Effect?](https://docs.monogame.net/articles/getting_to_know/whatis/graphics/WhatIs_ConfigurableEffect.html)*
- *[What Is a Render Target?](https://docs.monogame.net/articles/getting_to_know/whatis/graphics/WhatIs_Render_Target.html)*
- *[When does XNA discard Render Target contents?](https://stackoverflow.com/questions/6897889/when-does-xna-discard-render-target-contents)*
- *[What Is Sampler State?](https://docs.monogame.net/articles/getting_to_know/whatis/graphics/WhatIs_Sampler.html)*
