---
title: "unity2d游戏开发日志#1"
date: 2026-07-07T14:00:00+08:00
tags: ["日志", "游戏开发", "unity"]
categories: ["游戏开发日志"]
---

声明：这篇以及该系列仅为个人从0自学unity开发游戏日志，所以文章内容更多以记录为主

首先，我创建了两个对象，一个命名ground，一个命名player，用于表示地面和玩家，调整一下ground的大小以及位置，然后我们添加组件，给地面加上碰撞箱，不然玩家落下的时候会穿过去，玩家则是刚体以及胶囊碰撞箱

![展示图1](/images/ep2/展示图1.png)

![展示图2](/images/ep2/展示图2.png)

做完这些，我们点击播放

![展示图3](/images/ep2/展示图3.png)

ok，player物块顺利掉落到ground上

现在开始考虑一下移动，所以新建了一个script文件夹，在里面新建了一个名为Player\_Controller的c#脚本

![展示图4](/images/ep2/展示图4.png)

然后来编写一下移动脚本，众所周知，移动会改变对象的position，所以只要监控一下键位状态来调整position位置应该就ok了，这是我写的一个基础的c#脚本，有点编程基础的应该看得懂我在写啥，这里就不解释了

![展示图5](/images/ep2/展示图5.png)

但是第一个问题就出现了，角色移动的非常快，点一下马上就飞出屏幕了，对于一个游戏来说，这肯定是无法玩的，所以我打算先造两堵墙，然后慢慢调整单次触发方法的移动速度

![展示图6](/images/ep2/展示图6.png)

然后就出现了第二个问题

![展示图7](/images/ep2/展示图7.png)

然后我就去翻了一下入门教程并问了ai，确认了问题出现在我直接用 transform.position 移动角色，导致角色物理行为被忽略

简单来说，直接给transform.position传参并非我们熟悉的移动，更类似于传送，它不经过物理引擎，所以角色会直接穿过墙体。通过 Rigidbody2D进行的移动，才会触发碰撞检测

于是我用ai修改了一下脚本代码

![展示图8](/images/ep2/展示图8.png)

此时就能正常移动了(记得在 Rigidbody2D 组件里，找到 Constraints 部分，勾选 Freeze Rotation Z，不然方块会像轮子那样滚着走，别问我怎么知道的)

接下来就是完善冲刺以及跳跃，这里我为了加速工作流就用ai了，大概的代码如下

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

        // 执行水平移动
        rb.velocity = new Vector2(moveInput * moveSpeed, rb.velocity.y);

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

需要说明的是，update和fixupdate的区别在于，update是每帧调用，fixupdate则是固定间隔调用，也就是说，如果把移动，跳跃这些操作逻辑写在update里很可能就会出现游戏帧率影响角色移动速度之类的问题，所以我个人建议不要把那些很依赖物理系统和需要稳定节奏的操作（比如移动速度、跳跃力度、加速度变化）写在 Update 里

回归记录，将上述脚本保存后，如果直接用当然可以用，但是会有一个bug，因为我还没创建GroundCheck子对象，所以会出现角色可以无限跳的问题，这里我添加了这个子对象，命名为checkground，并把它的位置拖到地面上用于检测地面

![展示图9](/images/ep2/展示图9.png)

然后在player下，把玩家控制器脚本组件的ground check设置成我刚创建的GroundCheck子对象

![展示图10](/images/ep2/展示图10.png)

当然，这样还不行，我们需要让脚本需要知道哪些物体是地面，哪些不是，这里需要图层（Layer）来区分

我们在在 Inspector 窗口的最右上方，找到 Layer 下拉菜单，默认通常是 Default

点击它并选择Add Layer，我打了中文语言包，这里是添加图层

在弹出的列表中，找一个空行，输入名字：Ground

![展示图11](/images/ep2/展示图11.png)

点击地面物体，把它的Layer修改为刚才创建的Ground

![展示图12](/images/ep2/展示图12.png)

回到 Player 物体的脚本组件上，把 Ground Layer 下拉菜单也选择为 Ground

![展示图13](/images/ep2/展示图13.png)

然后就搞定了

![展示.gif](/images/ep2/展示.gif)
