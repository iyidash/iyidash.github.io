

<h1>简易水波纹（不使用RenderTexture和粒子系统）</h1>



> 简易水波的生成只需要 ***中心位置*** 信息即可，其余美术效果可以在shader里实现。
>
> 由于shader不能暂存信息，所以需要脚本将角色位置信息传递过来
>
> （geometry shader 可以暂存信息？）aaaa

<ul>
    <li><a href="#introMethod">实现方式</a></li>
    <li><a href="#customfunction">Custom Function</a><ul><li><a href="#customfunctionref">Custom Function代码参考</a></li></ul></li>
	<li><a href="#script">Script</a><ul><li><a href="#scriptref">代码参考</a></li></ul></li>
    <li><a href="#demo">参考效果</a></li>
    <li><a href="#extend">拓展</a><ul><li><a href="#watersparkle">波光粼粼 Shadergraph</a></li><li><a href="#watersparkledemo">波光粼粼参考效果</a></li></ul></li>
</ul>




<h2 id="introMethod"><em>实现方式：</em></h2>

1. 将角色位置以一定时间间隔储存到一个数组中 <span style="background-color: #bbe9ff">Vector4 waterRipples[]</span>

2. 将单个水波纹存活时间储存在 <span style="background-color: #bbe9ff">waterRipples.w</span> 中

   （时间最好需要考虑到角色是否移动，跳跃等）

3. 将 <span style="background-color: #bbe9ff">waterRipples</span> 传递给 Shader

4. 在shader里计算所有水波纹即可



<h2 id="customfunction"><em>Custom Fuction：</em></h2>

<p align="center">
  <img width="560" src="images\img_WaterInt\02_ripples-customfunction.png">
</p>


- 单个水波纹代码如下：

  ```c#
  float RipplesTransform(float time, float3 positionWS, float4 center, float radius = 3, float speed = 0.5, float noise = 0)
  {
      float dist = distance((center.xyz + float3(0, 0.5, 0)), (positionWS + noise) * float3(1, 1, 2));
      float changingRadius = center.w;
      changingRadius = 1 - changingRadius;
      changingRadius = 1 - pow(changingRadius,3);
      float opacity = sin(changingRadius * 3.1415926);
  
      float rings = abs(frac(dist - (time * speed)) - 0.5) * 2.0;
      rings  = pow(rings,4);
      float mask = RemapFloat01(dist, float2(0, radius * changingRadius));
      mask = 1.0 - mask;
      mask *= opacity;
  
      return rings * mask;
  }
  ```

- `dist`是到中心点的距离，其中`float3(0, .5, 0)`是中心点的高度偏移，用来调整水波中心位置的高度；`float3(1, 1, 2)`是用来缩放z方向上的距离，这样得到的就是一个椭圆形的水波（如果角色不在z轴为0的平面上的话不可以直接使用）

- `RemapFloat01()`将一个范围映射到01（自定义）

- `changingRadius`为一个0到1的值，用来控制半径变化以及透明度。

  半径大小的变化需要先快后慢（蓝色曲线），透明度也是需要先快后慢最后消失（橙色曲线）。

  <p align="center">
    <img width="300" src="images\img_WaterInt\01_graphic-centerX.png">
  </p>

- `rings`就是通过frac和abs来做循环

<p align="center">
  <img width="500" src="images\img_WaterInt\04_dist.gif">
</p>

- `void SetWaterRipples()`用来执行计算和输出结果到shadergraph。

  ```c#
  void SetWaterRipple_float(
      float3 posWS, float time, float noise, out float WaterRipples)
  {
      for (int i = 0; i < _ripplesParticlesSize; i++)
      {
          WaterRipples = lerp(RipplesTransform(time, posWS, _waterRipples[i], 7, 0.36, noise), 1, WaterRipples);
      }
  }
  ```


<p align="center">
  <img width="400" src="images\img_WaterInt\05_rings.gif">
</p>

<h4 id="customfunctionref">Custom Function参考代码如下：</h4>

```c#
//-----Unity ShaderGraph Custom Function-----

uniform float4 _waterRipples[10];
uniform int _ripplesParticlesSize;

float RemapFloat01(float In, float2 InMinMax)
{
    return clamp((In - InMinMax.x) / (InMinMax.y - InMinMax.x), 0, 1);
}

//generate according to center pos
float RipplesTransform(float time, float3 positionWS, float4 center, float radius = 3, float speed = 0.5, float noise = 0)
{
    float dist = distance((center.xyz + float3(0, 0.5, 0)), (positionWS + noise * .7) * float3(1, 1, 2));
    float changingRadius = center.w;
    changingRadius = 1 - changingRadius;
    changingRadius = 1 - pow(changingRadius,3);
    float opacity = sin(changingRadius * 3.1415926);

    float rings = abs(frac(dist - (time * speed)) - 0.5) * 2.0;
    rings  = pow(rings,4);
    float mask = RemapFloat01(dist, float2(0, radius * changingRadius));
    mask = 1.0 - mask;
    mask *= opacity;

    return rings * mask;
}

void SetWaterRipple_float( float3 posWS, float time, float noise,

                    out float WaterRipples)
{
    for (int i = 0; i < _ripplesParticlesSize; i++)
    {
        WaterRipples = lerp(RipplesTransform(time, posWS, _waterRipples[i], 7, 0.36, noise), 1, WaterRipples);
    }
}

```



<h2 id="script"><em>Script：</em></h2>

- `ripplesParticlesSize`和`waterRipples[]`需要传入shader。其中`waterRipples[].xyz`代表水波的中心位置，`waterRipples[].w`代表水波的存活时间；

- `waterRipples[].w`可以被`time_dissolving` 和` time_jump`ing影响到，`time_jumping`由`time_grounded`和`time_air`决定。

  这样一来`waterRipples[].w`的值就可以大致用来表示角色在运动时、起跳时、在空中时以及落地时的不同状态。

<h4 id="scriptref">参考代码如下：</h4>

```csharp
//-----C# Script-----

public class SetWaterRipples : MonoBehaviour
{
    private static int ripplesParticlesSize = 9;
    private int count;
    private Vector4[] waterRipples = new Vector4[ripplesParticlesSize];
    private bool[] startSpreading = new bool[ripplesParticlesSize];
    private float[] time_ripple = new float[ripplesParticlesSize];

    [SerializeField] private float interval = .2f;
    private float duration;
    private float time;
    private float time_dissolving;
    private Transform playerTr;
    private Vector3 playerPos;
    private CharacterMovement _characterMovement;
    private Controller _controller;
    void Start()
    {
        playerTr = GameObject.FindWithTag("Player").transform;
        _characterMovement = playerTr.gameObject.GetComponent<CharacterMovement>();
        _controller = playerTr.gameObject.GetComponent<Controller>();
        
        duration = (ripplesParticlesSize) * interval;
        Shader.SetGlobalInt("_ripplesParticlesSize", ripplesParticlesSize);
        
        for (int i = 0; i < ripplesParticlesSize; i++)
        {
            startSpreading[i] = false;
        }
    }
    
    void Update()
    {
        playerPos = playerTr.position;
        Timer();

        if (time >= interval)
        {
            waterRipples[count] = playerPos + new Vector3(Random.Range(-.3f, .3f), 0f, Random.Range(-1f, 1f));
            startSpreading[count] = true;
            
            count++;
            if (count > ripplesParticlesSize-1)
            {
                count = 0;
            }
            time = 0f;
        }
        Shader.SetGlobalVectorArray("_waterRipples", waterRipples);
    }

    private bool isGrounded = false;
    private float time_grounded;
    private float time_air;
    private float time_jumping;
    void Timer()
    {
        time += Time.deltaTime;
        for (int i = 0; i < ripplesParticlesSize; i++)
        {
            if (startSpreading[i])
            {
                time_ripple[i] += Time.deltaTime;
              	//combination of ripple appreance
                waterRipples[i].w = Mathf.Clamp01(Mathf.InverseLerp(0f, duration, time_ripple[i])) * Mathf.Lerp(time_dissolving, 1f,
                    (1f - time_jump) * Mathf.Sin(Mathf.PI * (1f - Mathf.Pow((1f - time_jump), 3f)) ));
                if (time_ripple[i] >= duration)
                {
                    startSpreading[i] = false;
                    time_ripple[i] = 0f;
                }
            }
        }
        
        //appear when jumping
        isGrounded = _controller.State.IsGrounded;
        if (isGrounded)
        {
            time_air = 0f;
            
            time_grounded += Time.deltaTime * .6f;
            if (time_grounded >= 1f)
            {
                time_grounded = 1f;
            }
        }
        else
        {
            time_grounded = 0f;
            time_air += Time.deltaTime * 1f;
            if (time_air >= 1f)
            {
                time_air = 1f;
            }
        }
        time_jumping = Mathf.Lerp(time_grounded, 1f, time_air);
        
        //appear when moving
        if (_characterMovement._horizontalMovement != 0f)
        {
            time_dissolving += Time.deltaTime * 1f;
            if (time_dissolving >= 1f)
            {
                time_dissolving = 1f;
            }
        }
        else
        {
            time_dissolving -= Time.deltaTime * 0.3f;
            if (time_dissolving <= 0f)
            {
                time_dissolving = 0f;
            }
        }
    }
}

```

<h2 id="extend"><em>拓展:</em></h2>

> 水面添加波光粼粼效果

<h4 id="watersparkle">波光粼粼效果近处比较少，远处比较明显。</h4>

可以使用 `positionWS.z` 结合 `Screen Position`  最后添加噪波等完成

*（下面是屏幕空间近小远大参考）*

<p align="center">
  <img width="600" src="images\img_WaterInt\03_sparklecone.png">
</p>


<h4 id="watersparkledemo">波光效果：</h4>





<h2 id="finalsg"><em>Final Shadergraph：</em></h2>

<p align="center">
  <img width="800" src="images\img_WaterInt\00_SG-waterInt.png">
</p>
