---
title: MonoGame 範例 NeonShooter 之 6
categories: MonoGame
date: 2025-06-15 12:33:58
updated: 2025-06-15 12:33:58
tags:
---

這是官方範例 [NeonShooter](https://github.com/MonoGame/MonoGame.Samples/tree/3.8.2/NeonShooter) 的第六篇，本篇將完成以下部分：
1. 加入粒子效果
2. 加入拖尾效果
3. 加入爆炸效果

<!-- more -->
### 1. 加入粒子效果
粒子系統用來表現各種視覺效果，大量的粒子可以模擬出火焰、爆炸、煙霧等效果，由於可能會有成千上萬的粒子，我們必須注意粒子系統的效能避免遊戲產生卡頓或降低了 FPS，範例中主要有兩個地方需要注意：
1. `SpriteBatch` 的參數使用 `SpriteSortMode.Deferred`。
2. 粒子的容器實作了一個 class `CircularParticleArray` 用來高效的重複利用記憶體。

第一點很容易理解，`SpriteSortMode.Deferred` 會根據呼叫 `SpriteBatch.Draw` 的順序去繪製物件，避免大量的物件排序。

第二點就要來研究一下 `CircularParticleArray` 的內容，看一下範例提供的程式碼：

{% codeblock ParticleManager.cs lang:csharp %}
// Represents a circular array with an arbitrary starting point. It is useful for efficiently overwriting
// the oldest particles when the array gets full. Simply overwrite particleList[0] and advance Start.
private class CircularParticleArray
{
	private int start;
	public int Start
	{
		get { return start; }
		set { start = value % list.Length; }
	}

	public int Count { get; set; }
	public int Capacity { get { return list.Length; } }
	private Particle[] list;

	public CircularParticleArray() { }  // for serialization

	public CircularParticleArray(int capacity)
	{
		list = new Particle[capacity];
	}

	public Particle this[int i]
	{
		get { return list[(start + i) % list.Length]; }
		set { list[(start + i) % list.Length] = value; }
	}
}
{% endcodeblock %}

可以看到 `list` 是一個 `Particle` 的陣列，存取只透過 `[]` operator，變數 `start` 用來標記存取陣列的起始點，變數 `Count` 代表目前使用中的粒子數量，當需要建立一個粒子時，如果 `Count` 還小於 `Capacity` 就直接從陣列中取用，如果 `Count` 等於 `Capacity` 代表陣列中所有粒子都已經在使用了，那就從 `start` 的位置開始，把最舊的粒子作為最新的粒子使用，由於粒子只是用來做視覺效果，即使捨棄了一些粒子也不會影響遊戲性，只要更新 `start` 就可以在不更改陣列的情況下循環利用粒子。

其中有一個狀況，有些粒子的生命周期比較長，那就會導致在陣列中間的粒子需要提早回收，這時候需要將結束的粒子交換到陣列的尾端，因為在 C# 中 class 是 reference type，在陣列中存的是 object 的 reference，大小取決於 OS 只有 4bytes 或 8bytes，並不會有大量的資料交換產生，如此一來就已經是一個足夠高效的資料結構了。

了解以上兩點以後，就可以開始著手實作粒子系統了，首先定義 `Particle` class，`Duration` 代表粒子存在時間，`NormalizeTime` 代表正規化後的經過時間，`LengthMultiplier` 代表長度變形時的參數。

{% codeblock Particle.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

namespace NeonShooter
{
    public class Particle
    {
        public Texture2D Texture = null;
		public Vector2 Position = Vector2.Zero;
		public Vector2 Velocity = Vector2.Zero;
		public Color Tint = Color.White;
		public float Rotation = 0f;
		public Vector2 Origin = Vector2.Zero;
		public Vector2 Scale = Vector2.One;

		public float Duration = 1f;
		public float NormalizeTime = 0f;
		public float LengthMultiplier = 1f;
    }
}
{% endcodeblock %}

在範例中 `ParticleManager` 使用了模板，目的是當有其他類型的 `Particle` 時可以更換 `State` 的資料類型而不必寫一個新的 `ParticleManager`，這邊就簡化不使用模板，我們只需要一個 `ParticleManager`。

接著定義 `Update` 的行為，每一幀根據速度移動，同時隨著時間顏色漸漸變淡。

{% codeblock Particle.cs lang:csharp %}
public void Update ()
{
    Vector2.Add (ref Position, ref Velocity, out Position);

    float speed = Velocity.Length ();
    float alpha = Math.Min (1, Math.Min (NormalizeTime * 2, speed * 1f));
    alpha *= alpha;

    Tint.A = (byte)(255 * alpha);

	Rotation = MathF.Atan2 (Velocity.Y, Velocity.X);
}
{% endcodeblock %}

注意這裡使用了 `Vector2.Add`，參數都是 pass by reference 因此不會產生複製，這在大量的運算中可以減少一些開銷，`alpha` 再次乘以自己則是可以得到較為快速地遞減，視覺效果上會更好。

長度也要隨著時間變短，碰到邊界以後會反彈。

{% codeblock Particle.cs lang:csharp %}
public void Update ()
{
    //...

	Scale.X = LengthMultiplier * Math.Min (Math.Min (1f, 0.1f * speed + 0.1f), alpha);

	if (Position.X < 0)
	{
		Velocity.X = Math.Abs (Velocity.X);
	}
	else if (Position.X > Game1.Width)
	{
		Velocity.X = -Math.Abs (Velocity.X);
	}

	if (Position.Y < 0)
	{
		Velocity.Y = Math.Abs (Velocity.Y);
	}
	else if (Position.Y > Game1.Height)
	{
		Velocity.Y = -Math.Abs (Velocity.Y);
	}
}
{% endcodeblock %}

最後，速度也要隨著時間減慢，這裡若 `Velocity` 數值已經很小了就設為 `Vector2.Zero`。

> 當浮點數非常接近於 0 的時候，在 IEEE754 標準中會改變取值的方式，稱做次正規數，運算時會比正規數多上許多 clock cycle，在大量運算中可能會造成效能問題。

{% codeblock Particle.cs lang:csharp %}
public void Update ()
{
    //...

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

基本的粒子這樣就完成了，接著加入 `ParticleManager` 來管理粒子。

{% codeblock Particle.cs lang:csharp %}
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

namespace NeonShooter
{
    public class CircularParticleArray
	{
		private readonly List<Particle> m_Particles;
		private int m_Start;
		private int m_Count;

		public int Start
		{
			get { return m_Start; }
			set { m_Start = value % m_Particles.Count; }
		}

		public int Count
		{
			get { return m_Count; }
			set { m_Count = value; }
		}

		public int Capacity { get { return m_Particles.Count; } }

		public CircularParticleArray (int _capacity)
		{
			m_Particles = new (_capacity);

			for (int i = 0; i < _capacity; i++)
			{
				m_Particles.Add (new Particle ());
			}
		}

		public Particle this[int _index]
		{
			get { return m_Particles[(m_Start + _index) % m_Particles.Count]; }
			set { m_Particles[(m_Start + _index) % m_Particles.Count] = value; }
		}

		public void Swap (int _lhs, int _rhs)
		{
			(this[_rhs], this[_lhs]) = (this[_lhs], this[_rhs]);
		}
	}

    public static class ParticleManager
    {
        private static readonly CircularParticleArray m_Particles = new (1024 * 20);

        public static void Update ()
        {
            int removalCount = 0;

            for (int i = 0; i < m_Particles.Count; i++)
            {
                Particle particle = m_Particles[i];

                particle.Update ();

                particle.NormalizeTime += 1f / particle.Duration;
                if (particle.NormalizeTime >= 1f)
                {
                    m_Particles.Swap (removalCount, i);

                    removalCount++;
                }
            }

            m_Particles.Start += removalCount;
            m_Particles.Count -= removalCount;
        }

        public static void Draw (SpriteBatch _spriteBatch)
        {
            for (int i = 0; i < m_Particles.Count; i++)
            {
                Particle particle = m_Particles[i];
                _spriteBatch.Draw (particle.Texture, particle.Position, null, particle.Tint, particle.Rotation, particle.Origin, particle.Scale, SpriteEffects.None, 0);
            }
        }

        public static void CreateParticle (Texture2D _texture, Vector2 _position, Vector2 _velocity, Color _tint, Vector2 _scale, float _duration)
		{
			Particle particle;
			if (m_Particles.Count == m_Particles.Capacity)
			{
				particle = m_Particles[0];
				m_Particles.Start++;
			}
			else
			{
				particle = m_Particles[m_Particles.Count];
				m_Particles.Count++;
			}

			particle.Texture = _texture;
			particle.Position = _position;
			particle.Velocity = _velocity;
			particle.Tint = _tint;
			particle.Rotation = MathF.Atan2 (_velocity.Y, _velocity.X);
			particle.Origin = new Vector2 (_texture.Width / 2f, _texture.Height / 2f);
			particle.Scale = _scale;
			particle.Duration = _duration;
			particle.NormalizeTime = 0f;
		}
    }
}
{% endcodeblock %}

需要注意的是在 `Update` 中，範例是將已經結束的粒子一步一步的交換至尾端，這裡改成與頭部的粒子交換，全部粒子都更新完畢以後再更新 `Start` 跟 `Count`。

別忘了在 `Game1` 中加入 ParticleManager，由於後面粒子的貼圖需要疊加顯示所以 `SpriteBatch.Begin` 的參數要改成 `BlendState.Additive`，代表渲染時後面畫的顏色會與已渲染的顏色疊加。

{% codeblock Game1.cs lang:csharp %}
protected override void Update (GameTime _gameTime)
{
	//...

	EntityManager.Update ();
	EnemySpawner.Update ();
	ParticleManager.Update ();

	//...
}

protected override void Draw (GameTime _gameTime)
{
	//...

	m_SpriteBatch.Begin (SpriteSortMode.Deferred, BlendState.Additive);
	EntityManager.Draw (m_SpriteBatch);
	ParticleManager.Draw (m_SpriteBatch);
	m_SpriteBatch.End ();

	//...
}
{% endcodeblock %}

### 2. 加入拖尾效果
觀察拖尾效果可以發現主要有三條線，一條直線，兩條波狀線相交出現，而每個粒子除了本來的路線還會出現一點偏移。

在 `PlayerShip` 中加入產生拖尾效果的程式碼，當角色移動的時候要在尾部生成粒子效果，將鍵盤輸入的結果存在 `m_Velocity` 中，判斷是否正在移動，以移動的方向為基準來產生粒子。

{% codeblock PlayerShip.cs lang:csharp %}
public override void Update ()
{
  	//...

    m_Velocity = direction * 8;
    m_Position += m_Velocity;
    m_Position.X = float.Clamp (m_Position.X, Size.X / 2, (Game1.Width - Size.X / 2));
    m_Position.Y = float.Clamp (m_Position.Y, Size.Y / 2, (Game1.Height - Size.Y / 2));

    CreateExhaustFire ();

    //...
}

//...

private void CreateExhaustFire ()
{
}
{% endcodeblock %}

先和子彈一樣的方法計算出角度與位置，所有的粒子都將從這個點出發。

{% codeblock PlayerShip.cs lang:csharp %}
private void CreateExhaustFire ()
{
	if (m_Velocity.LengthSquared () > 0f)
	{
		float rotation = MathF.Atan2 (m_Velocity.Y, m_Velocity.X);
		Quaternion quaternion = Quaternion.CreateFromYawPitchRoll (0f, 0f, rotation);
		Vector2 position = Position + Vector2.Transform (new Vector2 (-25, 0), quaternion);
	}
}
{% endcodeblock %}

將速度向量反轉以後長度調整到 3 作為每個粒子的基礎速度，中間的粒子隨機加上一個速度，兩側的粒子可以增加垂直的向量速度，透過 Sin 函數和遊戲時間變化向量的長度就可以達到看起來是波狀了，最後也和中間的粒子一樣加上一些隨機的速度。

{% codeblock Game1.cs lang:csharp %}
private BloomComponent m_BloomComponent;

public GameTime GameTime { get; private set; }

//...

protected override void Update (GameTime _gameTime)
{
    GameTime = _gameTime;

    //...
}
{% endcodeblock %}

{% codeblock PlayerShip.cs lang:csharp %}
private void CreateExhaustFire ()
{
    if (m_Velocity.LengthSquared () > 0f)
    {
        //...

        Vector2 baseVelocity = m_Velocity;
        baseVelocity.Normalize ();
        baseVelocity *= -3;

        const float alpha = 0.7f;

        double theta = m_Random.NextDouble () * 2 * Math.PI;
        float length = m_Random.NextSingle ();
        Vector2 middleVelocity = baseVelocity + length * new Vector2 ((float)Math.Cos (theta), (float)Math.Sin (theta));
        ParticleManager.CreateParticle (Art.LineParticle, position, middleVelocity, Color.White * alpha, new Vector2 (0.5f, 1), 60f);
        ParticleManager.CreateParticle (Art.Glow, position, middleVelocity, new Color (255, 187, 30) * alpha, new Vector2 (0.5f, 1), 60f);

        double time = Game1.Instance.GameTime.TotalGameTime.TotalSeconds;
        Vector2 perpVel = new Vector2 (baseVelocity.Y, -baseVelocity.X) * (0.6f * (float)Math.Sin (time * 10d));

        theta = m_Random.NextDouble () * 2 * Math.PI;
        length = m_Random.NextSingle () * 0.3f;
        Vector2 sideVelocity1 = baseVelocity + perpVel + length * new Vector2 ((float)Math.Cos (theta), (float)Math.Sin (theta));
        Vector2 sideVelocity2 = baseVelocity - perpVel + length * new Vector2 ((float)Math.Cos (theta), (float)Math.Sin (theta));
        ParticleManager.CreateParticle (Art.LineParticle, position, sideVelocity1, Color.White * alpha, new Vector2 (0.5f, 1), 60f);
        ParticleManager.CreateParticle (Art.LineParticle, position, sideVelocity2, Color.White * alpha, new Vector2 (0.5f, 1), 60f);
        ParticleManager.CreateParticle (Art.Glow, position, sideVelocity1, new Color (200, 38, 9) * alpha, new Vector2 (0.5f, 1), 60f);
        ParticleManager.CreateParticle (Art.Glow, position, sideVelocity2, new Color (200, 38, 9) * alpha, new Vector2 (0.5f, 1), 60f);
    }
}
{% endcodeblock %}

### 3. 加入爆炸效果
爆炸效果比較簡單，只要隨機方向撒出大量粒子就可以了，顏色的部分使用 HSV 的格式來隨機選取色相，可以較容易控制明度和彩度。

先加入 `ColorUtil` 來幫助使用，具體運算過程可以參考 wiki，這邊直接跳過。

{% codeblock ColorUtil.cs lang:csharp %}
using Microsoft.Xna.Framework;
using System;

namespace NeonShooter
{
    public class ColorUtil
    {
        public static Color HSVToColor (float h, float s, float v)
        {
            if (h == 0 && s == 0)
            {
                return new Color (v, v, v);
            }

            float c = s * v;
            float x = c * (1 - Math.Abs (h % 2 - 1));
            float m = v - c;

            return h switch
            {
                < 1 => new Color (c + m, x + m, m),
                < 2 => new Color (x + m, c + m, m),
                < 3 => new Color (m, c + m, x + m),
                < 4 => new Color (m, x + m, c + m),
                < 5 => new Color (x + m, m, c + m),
                _ => new Color (c + m, m, x + m)
            };
        }
    }
}
{% endcodeblock %}

在 `Enemy` 的 `Kill` 中加入粒子，隨機選取一個顏色，再隨機改變一點色相作為第二個顏色，每個粒子就從這兩個顏色中間隨機選取插值作為顏色。

另外在播放音效時也隨機改變一點頻率聽起來較為豐富。

{% codeblock Enemy.cs lang:csharp %}
public void Kill ()
{
    //...

    float hue1 = m_Random.NextSingle () * 6f;
    float hue2 = (hue1 + m_Random.NextSingle () * 2f) % 6f;
    Color color1 = ColorUtil.HSVToColor (hue1, 0.5f, 1);
    Color color2 = ColorUtil.HSVToColor (hue2, 0.5f, 1);

    for (int i = 0; i < 120; i++)
    {
        double theta = m_Random.NextDouble () * 2 * Math.PI;
		float speed = 18f * (1f - 1 / (m_Random.NextSingle () * 9f + 1f));
		Vector2 velocity = new (speed * (float)Math.Cos (theta), speed * (float)Math.Sin (theta));
		Color color = Color.Lerp (color1, color2, m_Random.NextSingle ());
		ParticleManager.CreateParticle (Art.LineParticle, Position, velocity, color, new Vector2 (1.5f, 1.5f), 190f);
    }

    Sound.Explosion.Play (0.5f, m_Random.NextSingle () * 0.4f - 0.2f, 0);
}
{% endcodeblock %}

最後在 `Bullet` 的 `Kill` 中也加入粒子。

{% codeblock Bullet.cs lang:csharp %}
public override void Update ()
{
	//...

	if (m_Position.X < 0 || m_Position.X > Game1.Width || m_Position.Y < 0 || m_Position.Y > Game1.Height)
	{
		//m_IsExpired = true;
		Kill ();
	}
}

public void Kill ()
{
    static readonly Random m_Random = new ();

	//...

	public void Kill ()
	{
		//...

		for (int i = 0; i < 30; i++)
		{
			double theta = m_Random.NextDouble () * 2 * Math.PI;
			float speed = m_Random.NextSingle () * 9f + 1f;
			Vector2 velocity = new (speed * (float)Math.Cos (theta), speed * (float)Math.Sin (theta));
			ParticleManager.CreateParticle (Art.LineParticle, Position, velocity, Color.LightBlue, Vector2.One, 50f);
		}
	}
}
{% endcodeblock %}

如此粒子效果就都添加完成了，下一篇預計將加入黑洞。

<video controls loop>
    <source src="/blog/videos/monogame-neon-shooter-6.mp4" type="video/mp4">
</video>

[NeonShooter](https://github.com/eyilee/MonoGame.Samples/tree/neon-shooter-6/NeonShooter)

***
參考資料
- *[Subnormal_number](https://en.wikipedia.org/wiki/Subnormal_number#Performance_issues)*
