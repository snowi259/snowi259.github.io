---
layout: post
title: "初次复活停服安卓游戏实操"
date: 2026-11-16
excerpt: "记一次停服游戏重新运行的过程。"
tags: [android]
comments: false
---

其实blog已经两年没更新了，一直都是在扒别人的开源项目

这个复活安卓项目从去年就开始了，只不过当时受限于大量的加密jsc，导致解包后一直没有进展

## 第一步是什么

复活停服游戏网上主要通过两个步骤：

1.重写服务端，也就是通常说的搭建私服，客户端

2.客户端处理之前所有的消息，也就是重写整个客户端之前发送服务器消息的所有逻辑，也就是制作成离线单机版

通常的第一步是找到原有apk解包，使用apktool解包分析游戏源文件

```powershell
java -jar apktool.jar d original.apk -o game
```

## 解包后的流程

这里使用的示例游戏是一款多年前的安卓轻量坦克大战网游

包内结构主要是大量jsc加密文件的cocos2d引擎游戏

## 分步骤分别修复最后整合

这次修复主要是分成不同块来修复

1.先修复正常进入大厅的游戏启动逻辑

我把启动流程接到本地存档上。读完玩家数据，再预加载大厅资源，最后切进大厅。下面是 InitScene.js 的几处改动。

```js
offlineProfile.initialize();
this._installOfflineLobbyHooks();

loading.LoadingScene.preload(lobbyResources, this._enterOfflineLobby.bind(this));

this._offlineBootDone = true;
game.start = true;
game.init();
cc.director.replaceScene(new mainInterface.GameScene());
```

2.修复正常游戏运行逻辑

解包发现游戏内不包含游戏地图，但是包含所有物块的框架和地图缩略图，通过codex进一步识别并批量还原

游戏的移动，攻击，伤害判定均由服务器计算，修改为单机版需批量重写

移动按每帧实际经过的秒数算步长。这里截的是朝右、朝左、朝上的部分，算完还要检查碰撞。

```js
var tileSize = Math.min(Number(tank.ITEM_WIDTH) || 48, Number(tank.ITEM_HEIGHT) || 48);
var distance = Math.min(speed * seconds, tileSize * 2);
var dx = 0;
var dy = 0;
if (dir === tank.Direction.RIGHT) {
    dx = distance;
} else if (dir === tank.Direction.LEFT) {
    dx = -distance;
} else if (dir === tank.Direction.UP) {
    dy = distance;
}
```

普攻和技能在单机房里直接交给坦克对象。先转向，再发射。

```js
var originalSendUseSkill = timelineProto.sendUseSkill;
timelineProto.sendUseSkill = function (dir, skillId) {
    if (this.roomInfo && this.roomInfo.alonePlayer) {
        var delegate = activateLocalControls(this.roomModel);
        if (delegate && !delegate.model._markForDelete) {
            var facing = isTankDirection(dir) ? dir : delegate.model.dir;
            if (isTankDirection(facing) && delegate.model.dir !== facing) {
                delegate.turnAround(facing);
            }
            delegate.launchBullet(skillId);
        }
        return;
    }
    return originalSendUseSkill.apply(this, arguments);
};

```

命中后调用游戏自己的扣血方法，死亡就记击杀。

```js

target.model.beHurt(cmd[3], cmd[6], cmd[1]);
if (target.model._markForDelete && !target.model.hasRecodeKill) {
    target.model.hasRecodeKill = true;
    tank.HurtManager.addkill(cmd[1], cmd[2]);
}
```

好消息是游戏内置了匹配模式的ai，不同坦克都内置了不同的ai逻辑(地图你不放本地，你把人机玩家放本地？)

## 总结

codex还是太好用了