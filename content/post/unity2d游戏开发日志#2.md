---
title: "unity2d游戏开发日志#2"
date: 2026-07-08T14:00:00+08:00
tags: ["日志", "游戏开发", "unity"]
categories: ["游戏开发日志"]
---


## 卡墙 Bug 的分析

直入主题，本期解决角色贴墙bug，导入角色精灵图


首先，需要先要理解为什么会卡住，具体表现为，在接触障碍物(比如墙面)且不在地面的时候，一直按住移动键角色会黏在障碍物上。


![1.png](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/1.png)


我用ai写的那份角色控制器脚本中，核心的移动代码为

rb.velocity = new Vector2(moveInput * moveSpeed, rb.velocity.y);


这样固然是标准的，那为什么会卡住呢？我个人的理解是这段代码每帧都在强行给 Rigidbody2D 赋值水平速度。当角色撞墙时，物理引擎阻止它穿墙，但下一帧代码又试图把它推进去。结果是，物理引擎和代码在‘推’和‘挡’之间互相拉扯，导致角色被卡在墙上，看起来像粘住了一样。就像一个人用身体顶着门，外面有人不断推他——他出不去，但也不停地感觉到力


简单来讲就是碰到障碍物了，速度变为0，下一帧移动代码又把速度改回去了，就这样一直循环


这个bug的解决方法我认为应该看游戏类型，我个人想要设计的是动作游戏，很多动作游戏（死亡细胞，武士零等）都是有挂在墙上的功能的，而且能够拓展出蹬墙跳之类的功能，这个时候就不能简单地一刀切。但是如果是一款经典平台跑酷游戏（比如马里奥），这个时候卡墙就很明显是个必须解决的bug


基于此，我构思了两套解决方案


1.针对想把贴墙作为机制的动作游戏，如果角色长时间（这个时间是一个完整的跳跃到落地所需的时间，超过就判定为长时间）脱离了地面并且x轴位置没变则开始计时，时间到0.5禁止该方向的水平移动直到落地


2.针对不打算把贴墙作为游戏机制的游戏，设置一个标签，为所有可能触发这个bug的地方贴上这个标签，然后在角色控制器里判断，脱离地面且碰到标签就停止该方向的水平移动（这个是我简单构思的不保证不会出现其他bug）


## 卡墙解决方案

那现在，开始解决，思路已经有了，这里我用ai加速工作流
```csharp
using System.Collections;
using UnityEngine;
public class Player : MonoBehaviour
{
    [Header("移动设置")]
    public float moveSpeed = 5f;
    private float moveInput;
    private float facingDirection = 1f;
    [Header("跳跃设置")]
    public float jumpForce = 12f;
    public Transform groundCheck;       // 地面检测点
    public LayerMask groundLayer;       // 地面图层
    private bool isGrounded;
    private bool jumpRequested;
    [Header("冲刺/翻滚设置")]
    public float dashSpeed = 15f;
    public float dashDuration = 0.2f;
    public float dashCooldown = 1f;
    private bool canDash = true;
    private bool isDashing;
    public bool IsInvincible { get; private set; }
    [Header("卡墙检测")]
    [Tooltip("一次完整跳跃从离地到落地的大概时间，超过这个时间且X轴没动则判定为贴墙")]
    public float fullJumpDuration = 2.0f;
    [Tooltip("贴墙后等待多久才禁止向墙方向移动")]
    public float wallStuckGraceTime = 0.5f;
    private float airborneTimer = 0f;       // 累计空中时间
    private float wallStuckTimer = 0f;      // 贴墙计时
    private float lastFrameX;               // 上一帧的X坐标
    private bool isWallBlocked;             // 是否已触发卡墙保护
    private int wallBlockedDir;             // 被阻挡的方向（1=右墙, -1=左墙, 0=无）
    private Rigidbody2D rb;
    private float originalGravity;
    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
        originalGravity = rb.gravityScale;
    }
    void Update()
    {
        if (isDashing) return;
        moveInput = Input.GetAxisRaw("Horizontal");
        
        if (moveInput != 0)
        {
            facingDirection = Mathf.Sign(moveInput);
        }
        // 只有在配置了 groundCheck 时，且在地面上才允许跳跃
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            jumpRequested = true;
        }
        if (Input.GetKeyDown(KeyCode.LeftShift) && canDash)
        {
            StartCoroutine(DashRoutine());
        }
    }
    void FixedUpdate()
    {
        if (isDashing) return;
        // 【防错处理】只有拖入了 groundCheck，才执行地面检测。否则默认在地面上。
        if (groundCheck != null)
        {
            isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.2f, groundLayer);
        }
        else
        {
            isGrounded = true;
        }
        // ========== 卡墙检测与保护 ==========
        float xDelta = Mathf.Abs(transform.position.x - lastFrameX);
        if (isGrounded)
        {
            // 落地重置所有卡墙状态
            airborneTimer = 0f;
            wallStuckTimer = 0f;
            isWallBlocked = false;
            wallBlockedDir = 0;
        }
        else
        {
            airborneTimer += Time.fixedDeltaTime;
            // 条件：空中时间超过完整跳跃周期 + X轴几乎没动 + 玩家在朝墙方向推
            bool pressingAgainstWall = moveInput != 0 && xDelta < 0.005f;
            bool airborneTooLong = airborneTimer > fullJumpDuration;
            if (pressingAgainstWall && airborneTooLong)
            {
                wallStuckTimer += Time.fixedDeltaTime;
                if (wallStuckTimer > wallStuckGraceTime)
                {
                    isWallBlocked = true;
                    wallBlockedDir = moveInput > 0 ? 1 : -1;
                }
            }
            else if (!pressingAgainstWall)
            {
                // 玩家换了方向或没按方向键，重置贴墙计时（但仍然在空中）
                wallStuckTimer = 0f;
            }
        }
        lastFrameX = transform.position.x;
        // ========== 执行水平移动（卡墙保护时禁止向墙方向移动） ==========
        float appliedMove = moveInput * moveSpeed;
        if (isWallBlocked)
        {
            // 禁止向墙方向继续输入，但允许反向离开墙壁
            if (moveInput > 0 && wallBlockedDir == 1)
                appliedMove = 0f;
            else if (moveInput < 0 && wallBlockedDir == -1)
                appliedMove = 0f;
            // 如果玩家转向了（朝远离墙的方向移动），解除卡墙保护
            if (moveInput != 0 && Mathf.Sign(moveInput) != wallBlockedDir)
            {
                isWallBlocked = false;
                wallBlockedDir = 0;
                wallStuckTimer = 0f;
            }
        }
        rb.velocity = new Vector2(appliedMove, rb.velocity.y);
        if (jumpRequested)
        {
            rb.velocity = new Vector2(rb.velocity.x, jumpForce);
            jumpRequested = false;
        }
    }
    private IEnumerator DashRoutine()
    {
        canDash = false;
        isDashing = true;
        IsInvincible = true;
        float originalGravity = rb.gravityScale;
        rb.gravityScale = 0f;
        rb.velocity = new Vector2(facingDirection * dashSpeed, 0f);
        yield return new WaitForSeconds(dashDuration);
        rb.gravityScale = originalGravity;
        rb.velocity = new Vector2(0f, rb.velocity.y);
        isDashing = false;
        IsInvincible = false;
        yield return new WaitForSeconds(dashCooldown);
        canDash = true;
    }
    private void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.red;
            Gizmos.DrawWireSphere(groundCheck.position, 0.2f);
        }
    }
}
```

## 配置墙检测

然后我们新建一个wallCheck对象放在player下，位置放在角色前一点

（没招了图片丢失了后期截图的将就看一下）
![2.png](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/20260708194801919.png)

然后就修复了这个bug

## 导入角色精灵图

接下来就是导入角色精灵图


我在itch.io网站上找了一张完整的史莱姆跳跃落下的图，暂时先用着

![史莱姆.png](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/New%20Piskel.png)


目前还是一整张，我们需要把每个动作单独拆分出来

按照图片这样设置，点击sprite editor，弹出来提示直接允许即可


![111](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/~1%404%299KD1QX8%25I~4%6037%24O~K.png)


自动然后切片即可，微调一下需要的精灵图范围，调到自己满意的水平，然后应用


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/5HNLN8B7%5D_RW_I%7BM5_9KWL7.png)


然后精灵图就被自己切好了

![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/20260708194801919.png)


将需要作为站立帧的精灵图拖进player的sprite renderer组件，然后把胶囊碰撞箱的方向从垂直设置为水平并微调一下让碰撞箱和角色轮廓贴合


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/2.png)
![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/%7BOCABE6YHK%405%29F5PQK~XZ7Y.png)
![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/3.png)

然后微调一些子对象的位置


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/4.png)


## 配置动画

接下来就是动画


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/I6D63%25EN%402O%5DM%29F_G~GK7OI.png)


创建一个动画文件放在你找得到的地方


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/38NZB%25SG%5B%25Z%28B~%5B%25BTP~X%5DE.png)


当然我更建议把起跳，下落，落地分为三段单独的动画而不是一整段

![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/ZGMK9~M9KXY%25%40LC%24JU1%7BG%5DY.png)


这里说一下，你可能会遇到关掉动画窗口后直接打开动画文件发现不能播放的问题，其实只需要先选中player组件然后在编辑器左上方的窗口界面打开动画即可


言归正传，现在是往动画里插入精灵图，游戏里的角色动起来本质上就是一套精灵图循环播放


好吧其实没那么简单，先一步一步来，先把需要的精灵图拖进动画里，调整到合适的速率


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/%60EC%5BZC2%7DPQ2Z%24E15RNC%5DC6E.png)


然后这个时候第一个问题就来了，我们很明显是需要起跳后达到最高点持续展示一张下落的精灵图，不能简单地只是一套精灵图循环，这个时候三套动画的好处就显现出来了


大概思路为，起跳后—>播放jump\_up动画—>判断是否达到最高点—>达到最高点后播放jump\_down动画(通常就一张图)—>判断落地—>落地后播放touch\_ground动画


配置完动画后，搭一下动画器，本质是个条件状态机


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/K0%28_%7BC%29FK5WA4UOBYKE~L%29G.png)


## 修改脚本：接入动画

然后我们需要改一下脚本，首先添加动画相关的字段，然后在 Start() 中获取 Animator 组件，最后在跳跃执行处添加 Jump 触发器，这就是大概思路，我依旧ai


```csharp
using System.Collections;
using UnityEngine;
public class Player : MonoBehaviour
{
    [Header("移动设置")]
    public float moveSpeed = 5f;
    private float moveInput;
    private float facingDirection = 1f;
    [Header("跳跃设置")]
    public float jumpForce = 12f;
    public Transform groundCheck;       // 地面检测点
    public LayerMask groundLayer;       // 地面图层
    private bool isGrounded;
    private bool jumpRequested;
    [Header("冲刺/翻滚设置")]
    public float dashSpeed = 15f;
    public float dashDuration = 0.2f;
    public float dashCooldown = 1f;
    private bool canDash = true;
    private bool isDashing;
    public bool IsInvincible { get; private set; }
    [Header("卡墙检测")]
    [Tooltip("一次完整跳跃从离地到落地的大概时间，超过这个时间且X轴没动则判定为贴墙")]
    public float fullJumpDuration = 2.0f;
    [Tooltip("贴墙后等待多久才禁止向墙方向移动")]
    public float wallStuckGraceTime = 0.5f;
    private float airborneTimer = 0f;       // 累计空中时间
    private float wallStuckTimer = 0f;      // 贴墙计时
    private float lastFrameX;               // 上一帧的X坐标
    private bool isWallBlocked;             // 是否已触发卡墙保护
    private int wallBlockedDir;             // 被阻挡的方向（1=右墙, -1=左墙, 0=无）
    [Header("动画设置")]
    public Animator animator;
    private bool \_wasGrounded;
    private bool \_wasRising;
    private Rigidbody2D rb;
    private float originalGravity;
    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
        originalGravity = rb.gravityScale;
        if (animator == null)
            animator = GetComponent<Animator>();
    }
    void Update()
    {
        if (isDashing) return;
        moveInput = Input.GetAxisRaw("Horizontal");
        
        if (moveInput != 0)
        {
            facingDirection = Mathf.Sign(moveInput);
        }
        // 只有在配置了 groundCheck 时，且在地面上才允许跳跃
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            jumpRequested = true;
        }
        if (Input.GetKeyDown(KeyCode.LeftShift) && canDash)
        {
            StartCoroutine(DashRoutine());
        }
    }
    void FixedUpdate()
    {
        if (isDashing) return;
        // 【防错处理】只有拖入了 groundCheck，才执行地面检测。否则默认在地面上。
        if (groundCheck != null)
        {
            isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.2f, groundLayer);
        }
        else
        {
            isGrounded = true;
        }
        // ========== 卡墙检测与保护 ==========
        float xDelta = Mathf.Abs(transform.position.x - lastFrameX);
        if (isGrounded)
        {
            // 落地重置所有卡墙状态
            airborneTimer = 0f;
            wallStuckTimer = 0f;
            isWallBlocked = false;
            wallBlockedDir = 0;
        }
        else
        {
            airborneTimer += Time.fixedDeltaTime;
            // 条件：空中时间超过完整跳跃周期 + X轴几乎没动 + 玩家在朝墙方向推
            bool pressingAgainstWall = moveInput != 0 && xDelta < 0.005f;
            bool airborneTooLong = airborneTimer > fullJumpDuration;
            if (pressingAgainstWall && airborneTooLong)
            {
                wallStuckTimer += Time.fixedDeltaTime;
                if (wallStuckTimer > wallStuckGraceTime)
                {
                    isWallBlocked = true;
                    wallBlockedDir = moveInput > 0 ? 1 : -1;
                }
            }
            else if (!pressingAgainstWall)
            {
                // 玩家换了方向或没按方向键，重置贴墙计时（但仍然在空中）
                wallStuckTimer = 0f;
            }
        }
        lastFrameX = transform.position.x;
        // ========== 执行水平移动（卡墙保护时禁止向墙方向移动） ==========
        float appliedMove = moveInput * moveSpeed;
        if (isWallBlocked)
        {
            // 禁止向墙方向继续输入，但允许反向离开墙壁
            if (moveInput > 0 && wallBlockedDir == 1)
                appliedMove = 0f;
            else if (moveInput < 0 && wallBlockedDir == -1)
                appliedMove = 0f;
            // 如果玩家转向了（朝远离墙的方向移动），解除卡墙保护
            if (moveInput != 0 && Mathf.Sign(moveInput) != wallBlockedDir)
            {
                isWallBlocked = false;
                wallBlockedDir = 0;
                wallStuckTimer = 0f;
            }
        }
        rb.velocity = new Vector2(appliedMove, rb.velocity.y);
        if (jumpRequested)
        {
            rb.velocity = new Vector2(rb.velocity.x, jumpForce);
            jumpRequested = false;
            if (animator != null)
                animator.SetTrigger("Jump");
        }
        // ========== 动画状态更新 ==========
        if (animator != null)
        {
            animator.SetBool("isGrounded", isGrounded);
            animator.SetFloat("velocityY", rb.velocity.y);
            // 检测落地 → 触发 touch\_ground
            if (isGrounded && !\_wasGrounded)
            {
                animator.SetTrigger("Land");
            }
            // 检测从边缘掉落（没跳跃就离地，直接进入下落动画）
            if (!isGrounded && \_wasGrounded && rb.velocity.y <= 0)
            {
                animator.SetTrigger("Fall");
            }
            \_wasGrounded = isGrounded;
            \_wasRising = rb.velocity.y > 0;
        }
    }
    private IEnumerator DashRoutine()
    {
        canDash = false;
        isDashing = true;
        IsInvincible = true;
        float originalGravity = rb.gravityScale;
        rb.gravityScale = 0f;
        rb.velocity = new Vector2(facingDirection * dashSpeed, 0f);
        yield return new WaitForSeconds(dashDuration);
        rb.gravityScale = originalGravity;
        rb.velocity = new Vector2(0f, rb.velocity.y);
        isDashing = false;
        IsInvincible = false;
        yield return new WaitForSeconds(dashCooldown);
        canDash = true;
    }
    private void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.red;
            Gizmos.DrawWireSphere(groundCheck.position, 0.2f);
        }
    }
}
```

然后配置一下动画器，先创建几个参数


isGrounded	Bool	每帧更新，是否在地面

velocityY	Float	每帧更新，垂直速度

Jump	Trigger	主动按跳跃键时触发一次

Fall	Trigger	没按跳跃就从边缘掉落时触发一次

Land	Trigger	落地瞬间触发一次


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/%28%24T04IOS%24%60BU73041DW_PD8.png)


## 动画器配置

然后我们需要创建一个idle动画，意味站立，用来展示默认状态


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/8UZRZYQ%25F5R4%7D~G4X%7D9B1BI.png)


点击设为图层默认状态(Set as Layer Default State)


然后idle处右键创建过渡


连成我这样子


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/W%24~9%7D%5B%296AP2V91%29BSXLZFTD.png)


然后给每条线加一些条件，不过先把每个动画的设置改成这样


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/P~Z2T%28EJB9TJO%28D%25AFOWS~F.png)


然后按照ai给的方案配置每条过渡的条件


![](https://cdn.jsdelivr.net/gh/brain584/imgonly@main/test/FB8G%402%7DP%24T6%28%5DWAYA~U6Q7U.png)


## 两个 Bug

然后就可以发现，有两个 bug：

1. 设置完动画后，点击 Play，角色起跳到最高处就完成所有动画了——起跳、下落、落地这一套流程。需要在 Play 过程中主动将接触地面和站立之间那条过渡的「有退出时间」关掉再打开才能正常。

2. 动画不是打断的，必须等上个动画播放完成后才播放下个动画。比如落地后马上起跳，必须要等接触地面的动画播放完成后才播放下一个动画，即使现在在空中。


针对bug2，我有一个思路，首先，我的接触地面的动画用的精灵图是包含了史莱姆落地时旁边的那两小坨史莱姆的，如果这个时候直接起跳那两小坨史莱姆也会飞天上，于是我计划原先接触地面的动画必须从高处落下才展示，观察角色跳跃时y轴的高差，单段跳跃高度大概为2，则落体处和地面高差如果大于3.5再触发接触地面的动画，否则触发新动画

（声明一下，我搞了快八个小时了真力竭了，我是一边学unity一边写博客的，现在真力竭了，下面内容均使用ai辅助生成）

## 解决 Bug 1：起跳到最高点就把所有动画播完了

首先排查 bug 1。把动画器里的过渡一条条看了半天，最后发现是代码的问题。

在 FixedUpdate 里，我是这样判断落地的：

```csharp
// 检测落地 → 触发 touch_ground
if (isGrounded && !_wasGrounded)
{
    animator.SetTrigger("Land");
}
```
问题出在 `_wasGrounded` 上。这个字段默认值是 `false`，而游戏一开始角色站在地面上，`isGrounded` 是 `true`。于是第一帧就满足了 `isGrounded && !_wasGrounded`，直接发了一个 `Land` trigger。

但这个时候动画器还在 Idle 状态，`Land` trigger 没有被任何过渡消费，就一直残留在动画器里。等到角色起跳 → 到达最高点 → 速度变为负 → 动画器切到 `jump_down` 的时候，这个残留的 `Land` trigger 还在，直接把 `jump_down → touch_ground` 的过渡触发了。

于是角色在最高点就把起跳、下落、落地三套动画全播完了。

修复很简单，把 `_wasGrounded` 初始值改成 `true` 就行了：

```csharp
private bool _wasGrounded = true;   // 初始 true，防止首帧误触发
```
这个 bug 折磨了我好一阵子，结果就一行代码的事。只能说 Trigger 机制好用是好用，但残留问题是真的坑。

## 解决 Bug 2：高落差重落地 + 低落差轻落地

然后是 bug 2。我的思路是：记录离地时的 Y 坐标，落地时算出下落高度，超过 3.5（大约比单段跳跃高一点）就播重落地动画，否则播轻落地动画。重落地动画必须播完才能动，轻落地可以随时被跳跃打断。

新增字段：

```csharp
[Header("落地动画")]
public float heavyLandThreshold = 3.5f;
private float _airborneStartY;       // 离地时的 Y 坐标
private bool _isLandingLocked;       // 重落地时锁定所有输入
private Coroutine _heavyLandRoutine; // 重落地协程引用
```
在动画状态更新里，离地时记录 Y 坐标，落地时算落差：

```csharp
// 离地瞬间 → 记录起跳 Y 坐标
if (!isGrounded && _wasGrounded)
{
    _airborneStartY = transform.position.y;
}
// 落地瞬间 → 根据落差播放不同动画
if (isGrounded && !_wasGrounded)
{
    float fallDistance = _airborneStartY - transform.position.y;
    if (fallDistance > heavyLandThreshold)
    {
        animator.SetTrigger("HeavyLand");
        _isLandingLocked = true;
        if (_heavyLandRoutine != null) StopCoroutine(_heavyLandRoutine);
        _heavyLandRoutine = StartCoroutine(HeavyLandRoutine());
    }
    else
    {
        animator.SetTrigger("LightLand");
    }
}
```
重落地时启动一个协程，等着动画播完再解锁：

```csharp
private IEnumerator HeavyLandRoutine()
{
    float timer = 0f;
    while (timer < 3.0f)
    {
        var state = animator.GetCurrentAnimatorStateInfo(0);
        if (state.IsName("touch_ground") && state.normalizedTime >= 1.0f)
            break;
        timer += Time.deltaTime;
        yield return null;
    }
    _isLandingLocked = false;
    animator.SetTrigger("LandDone");
    _heavyLandRoutine = null;
}
```
同时在 Update 里加了锁判断——锁住的时候直接 return，不处理任何输入；FixedUpdate 里也把水平移动置零。

## 又出问题了：高处落下卡死 + 延迟

写完跑了一下，高处落下后角色会先卡住好一会，然后才播重落地动画，播完动画器就卡死不动了。

排查了很久，发现是**竞态条件**。`HeavyLand` trigger 发出去之后，动画器需要至少一帧才能切到 `touch_ground` 状态。但我的解锁代码是这样的：

```csharp
if (!state.IsName("touch_ground") || state.normalizedTime >= 1.0f)
{
    _isLandingLocked = false;
    animator.SetTrigger("LandDone");
}
```
在还没切过去的那一帧，`!IsName("touch_ground")` 直接为 true，立刻就解锁了。`LandDone` trigger 也发出去了。等到动画器终于切到 `touch_ground` 的时候，一切都已经结束了——锁解了，trigger 也错过了，动画器就这么被晾在那里。

我试着加各种状态追踪来规避这个问题，但越改越复杂，该出问题还是出问题。

## 彻底放弃 Trigger，改用 Play() 直接切

这时候我突然意识到：我到底为什么在跟动画器的过渡系统较劲？

Trigger 机制的问题在于它是异步的——你发出 trigger，动画器不一定立刻响应，中间可能隔若干帧。这就导致了各种奇怪的竞态，尤其是在需要精确时序的跳跃/落地切换场景里。

所以我决定**完全绕过动画器的过渡系统**。不用 trigger，不连过渡线，直接用 `animator.Play("状态名")` 来切动画。Play() 是同步的，调用即生效，不存在任何中间态。

改完之后的 FixedUpdate 动画控制部分大概是这样的：

```csharp
// ========== 动画状态控制（全部用 Play 直接切，不依赖 Animator 过渡） ==========
if (animator != null)
{
    var curState = animator.GetCurrentAnimatorStateInfo(0);
    animator.SetBool("isGrounded", isGrounded);
    animator.SetFloat("velocityY", rb.velocity.y);
    // 离地瞬间 → 记录起跳 Y 坐标
    if (!isGrounded && _wasGrounded)
    {
        _airborneStartY = transform.position.y;
    }
    // 落地瞬间
    if (isGrounded && !_wasGrounded)
    {
        float fallDistance = _airborneStartY - transform.position.y;
        if (fallDistance > heavyLandThreshold)
        {
            animator.Play("touch_ground");
            _isLandingLocked = true;
            if (_heavyLandRoutine != null) StopCoroutine(_heavyLandRoutine);
            _heavyLandRoutine = StartCoroutine(HeavyLandRoutine());
        }
        else
        {
            animator.Play("light_land");
        }
    }
    // 边缘掉落 → 直接播 jump_down
    if (!isGrounded && _wasGrounded && rb.velocity.y <= 0)
    {
        animator.Play("jump_down");
    }
    // 检测达到最高点 → jump_up 切 jump_down
    if (!isGrounded && curState.IsName("jump_up") && _wasRising && rb.velocity.y <= 0.1f)
    {
        animator.Play("jump_down");
    }
    // 轻落地播完 → 自动回 Idle
    if (!_isLandingLocked && curState.IsName("light_land") && curState.normalizedTime >= 1.0f)
    {
        animator.Play("Idle");
    }
    _wasGrounded = isGrounded;
    _wasRising = rb.velocity.y > 0;
}
```
起跳的地方也改成 Play：

```csharp
if (jumpRequested)
{
    rb.velocity = new Vector2(rb.velocity.x, jumpForce);
    jumpRequested = false;
    if (animator != null)
        animator.Play("jump_up");
}
```
协程收尾也直接用 Play 切回 Idle：

```csharp
private IEnumerator HeavyLandRoutine()
{
    float timer = 0f;
    while (timer < 3.0f)
    {
        var state = animator.GetCurrentAnimatorStateInfo(0);
        if (state.IsName("touch_ground") && state.normalizedTime >= 1.0f)
            break;
        timer += Time.deltaTime;
        yield return null;
    }
    _isLandingLocked = false;
    animator.Play("Idle");
    _heavyLandRoutine = null;
}
```
## 极简动画器配置

改完之后，动画器的配置也极度简化了。不再需要那些复杂的过渡线：

- 创建 5 个状态：`Idle`、`jump_up`、`jump_down`、`touch_ground`、`light_land`
- 把对应的动画 clip 拖进去
- 每个空中状态的 Loop Time 关掉
- Idle 设为默认状态
- **不连任何过渡线**

对，你没看错——空中和落地状态之间一条过渡线都没有。所有切换都由代码里的 `animator.Play()` 直接完成。

这样做的好处：

1. **没有竞态**：Play() 是同步的，不会出现 trigger 残留的问题
2. **没有延迟**：不需要等 transition duration
3. **不会卡死**：不存在"发出了 trigger 但动画器没收到"的情况
4. **调试简单**：哪个状态不对，直接看代码在哪行 Play 的就行了

## 踩坑总结

这一轮折腾下来最大的收获是：**动画器的 Trigger 过渡系统适合做"松散"的状态切换（比如待机↔走路），但不适合需要精确帧时序的切换（跳跃、落地这类）。** 后者用代码直接 Play() 控制要可靠得多。

不是说 Trigger 不好——它在做复杂的混合树和条件过渡时非常强大——只是对于 2D 动作游戏这种每个动画切换都需要精确控制的场景，`animator.Play()` 省心太多了。

下一期应该开始搞战斗系统了。
