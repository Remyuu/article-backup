# Compute Shader学习笔记（三）之 物理模拟



查看之前的笔记：

[Compute Shader学习笔记（一）](https://zhuanlan.zhihu.com/p/699253914)

[Compute Shader学习笔记（二）之 后处理效果](https://zhuanlan.zhihu.com/p/700148560)

[Compute Shader学习笔记（二）之 粒子效果与群集行为模拟](https://zhuanlan.zhihu.com/p/700370323)

[Compute Shader学习笔记（三）之 草地渲染](https://zhuanlan.zhihu.com/p/701633578)



### 小试牛刀

上面源代码的Main分支就是初始工程，打开工程中的 `SimplePhysics` 场景。

![image-20240617135326685](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240617135326685.png)

项目很简单。在一个Cube的内部随机生成若干个球，在Compute Shader中利用上一帧的速度和位置等信息更新出下一帧的数据。最后用GPU Instancing把他们渲染出来。

首先将球钳制在Cube内。

```
// keep objects inside room
if ( ball.position.x < LIMITS_MIN_X || ball.position.x > LIMITS_MAX_X ) {
    ball.position.x = clamp( ball.position.x, LIMITS_MIN_X, LIMITS_MAX_X );
    ball.velocity.x = - ball.velocity.x;
}

if ( ball.position.y < LIMITS_MIN_Y ) {
    ball.position.y = LIMITS_MIN_Y;
    ball.velocity.xz *= 0.96;
    ball.velocity.y = - ball.velocity.y * 0.8;
}

if ( ball.position.z < LIMITS_MIN_Z || ball.position.z > LIMITS_MAX_Z ) {
    ball.position.z = clamp( ball.position.z, LIMITS_MIN_Z, LIMITS_MAX_Z );
    ball.velocity.z = - ball.velocity.z;
}
```

每一帧小球都会受到空气阻力和重力。

```
ball.velocity.xz *= 0.98;
ball.velocity.y -= 9.8 * deltaTime;
```

![image-20240618095139454](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240618095139454.png)

小球之间的碰撞检测。





```
float3 normal;
float3 relativeVelocity;

for ( int i = id.x + 1; i < ballsCount; i ++ ) {
    Ball ball2 = ballsBuffer[ (uint)i ];

    normal = ball.position - ball2.position;
    const float distance = length(normal);

    if ( distance < 2 * radius ) {
        normal *= 0.5 * distance - radius;
        ball.position -= normal;
        normal = normalize(normal);
        relativeVelocity = ball.velocity - ball2.velocity;
        normal *= dot( relativeVelocity, normal );
        ball.velocity -= normal;
    }
}
```