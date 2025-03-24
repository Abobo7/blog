---
title: 某手游反作弊(Xigncode)的研究与绕过
date: 2025/3/24


---



---

检测root、检测xposed，甚至检测代理的反作弊见的多了，检测应用列表的反作弊还真是头一回见，只要在它的黑名单里，就会直接触发反作弊，结果就是——游戏弹窗，然后自杀（如下图）

![img](https://nie.res.netease.com/r/pic/20240628/af154895-0355-41f7-84ed-71c2ee932f81.png)

这让身为搞机党的笔者十分头疼，毕竟谁设备上还没点hacker工具呢，稍不留神就上黑名单了。在尝试隐藏magisk(shamiko、kitsune mask+ magisk hide)、隐藏lsposed、屏蔽读取app名单（Lsp模块）均无果后，笔者感受到了深深的挫败感，毕竟没法读取这玩意的日志，鬼知道它到底检测到了什么。遂只能逆向一下apk，看看到底是怎么检测，以及如何触发闪退的咯。

先说结论，目前笔者只是移除了客户端的xc反作弊，但是官方服务端会收集客户端提供的token，因此绕过反作弊的客户端无法登录官服（悠星你坏事做尽）。至于绕过反作弊的客户端怎么用嘛，嘿嘿，那就要看各位读者大显神通了。

老规矩，咱先从Jvav层看起。官方客户端使用了xapk格式，只用提取主apk，使用jadx反编译一下

应用的主入口在`com.yostarjp.bluearchive.MxUnityPlayerActivity`，反编译代码如下：

```java
package com.yostarjp.bluearchive;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.DialogInterface;
import android.content.Intent;
import android.content.res.Configuration;
import android.os.Bundle;
import android.util.DisplayMetrics;
import android.view.WindowManager;
import com.airisdk.sdkcall.AiriSDK;
import com.amazon.identity.auth.device.datastore.AESEncryptionHelper;
import com.unity3d.player.UnityPlayer;
import com.unity3d.player.UnityPlayerActivity;
import com.wellbia.xigncode.XigncodeClient;
import com.wellbia.xigncode.XigncodeClientSystem;

/* loaded from: classes2.dex */
public class MxUnityPlayerActivity extends UnityPlayerActivity implements XigncodeClientSystem.Callback {
    @Override // com.wellbia.xigncode.XigncodeClientSystem.Callback
    public void OnLog(final String msg) {
    }

    @Override // com.wellbia.xigncode.XigncodeClientSystem.Callback
    public int SendPacket(byte[] buffer) {
        return 0;
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        AiriSDK.onCreate(this, savedInstanceState);
        boolean z = XigncodeClient.getInstance().initialize((Activity) this, "4v1cpbkiIB7t", "", (XigncodeClientSystem.Callback) this) == 0;
        UnityPlayer.UnitySendMessage("XIGNCODE3Interface", "OnInitialized", String.format("{\"succeeded\": %b}", Boolean.valueOf(z)));
        if (z) {
            return;
        }
        runOnUiThread(new Runnable() { // from class: com.yostarjp.bluearchive.MxUnityPlayerActivity.1
            @Override // java.lang.Runnable
            public void run() {
                AlertDialog.Builder builder = new AlertDialog.Builder(MxUnityPlayerActivity.this);
                builder.setTitle("Initialization Failed.");
                builder.setMessage("アプリケーションを正しく初期化できないため、終了します。");
                builder.setCancelable(false);
                builder.setPositiveButton("ok", new DialogInterface.OnClickListener() { // from class: com.yostarjp.bluearchive.MxUnityPlayerActivity.1.1
                    @Override // android.content.DialogInterface.OnClickListener
                    public void onClick(DialogInterface dialog, int id) {
                        dialog.cancel();
                    }
                });
                builder.create().show();
            }
        });
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity
    protected void onResume() {
        XigncodeClient.getInstance().onResume();
        super.onResume();
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity
    protected void onPause() {
        XigncodeClient.getInstance().onPause();
        super.onPause();
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity
    protected void onDestroy() {
        XigncodeClient.getInstance().cleanup();
        super.onDestroy();
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        AiriSDK.onNewIntent(this, intent);
    }

    @Override // com.unity3d.player.UnityPlayerActivity, android.app.Activity, android.content.ComponentCallbacks
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        WindowManager windowManager = (WindowManager) getSystemService("window");
        DisplayMetrics displayMetrics = new DisplayMetrics();
        if (isInMultiWindowMode()) {
            windowManager.getDefaultDisplay().getMetrics(displayMetrics);
        } else {
            windowManager.getDefaultDisplay().getRealMetrics(displayMetrics);
        }
        UnityPlayer.UnitySendMessage("GraphicsManager", "OnConfigChanged", displayMetrics.widthPixels + AESEncryptionHelper.SEPARATOR + displayMetrics.heightPixels);
    }

    @Override // com.wellbia.xigncode.XigncodeClientSystem.Callback
    public void OnHackDetected(int code, String info) {
        UnityPlayer.UnitySendMessage("XIGNCODE3Interface", "OnHackDetected", String.format("{\"code\": %d, \"info\": \"%s\"}", Integer.valueOf(code), info));
        runOnUiThread(new Runnable() { // from class: com.yostarjp.bluearchive.MxUnityPlayerActivity.2
            @Override // java.lang.Runnable
            public void run() {
                AlertDialog.Builder builder = new AlertDialog.Builder(MxUnityPlayerActivity.this);
                builder.setTitle("Use of unauthorized apps.");
                builder.setMessage("不明なアプリが検出されたため、終了します。");
                builder.setCancelable(false);
                builder.setPositiveButton("ok", new DialogInterface.OnClickListener() { // from class: com.yostarjp.bluearchive.MxUnityPlayerActivity.2.1
                    @Override // android.content.DialogInterface.OnClickListener
                    public void onClick(DialogInterface dialog, int id) {
                        dialog.cancel();
                    }
                });
                builder.create().show();
            }
        });
    }
}
```

可以看到，入口继承了`UnityPlayerActivity`这个类，这个是Unity的相关类，和反作弊没有关系，咱们先放一边；后面的才是咱们的重头戏。`implements XigncodeClientSystem.Callback`这里很显然表明入口实现了`XigncodeClientSystem`接口中的`Callback`，非常明显的反作弊好吧。

然后，在入口的`onCreate`方法里，先对`XigncodeClient`进行初始化，将初始化的结果发送给Unity，如果初始化成功了就退出`onCreate`，失败了就显示初始化失败的弹窗，相信读者应该都能看懂，对吧？

接下来的`onResume`、`onPause`、`onDestroy`均调用了Xigncode的对应回调函数。

在最后的`OnHackDetected`里，则是在检测到作弊时向Unity发送调用信息然后显示检测到作弊的弹窗。

看到这里，相信读者都和笔者一样，认为只要重写了`OnHackDetected`方法，不就能实现不让游戏自杀了吗？很遗憾，它只是一个回调函数，代码逻辑里也很清楚的展现了，这里并没有任何涉及app自杀的逻辑，因此直接修改这里是行不通的。那Unity层呢？经过笔者对il2cpp进行dump并跑完IDA脚本后，在Unity中存在`XIGNCODE3Interface`这样一个类，也有`OnHackDetected`的回调函数，实际上就是这里发送消息的调用对象，然而这个类也是没有执行自杀的相关逻辑的，因此笔者猜测应该是存在于xigncode自己的so中的，笔者并没有对这个so进行逆向，因此也无从证实了，读者若有看法可于评论区一同探讨。

到这里很明显只剩下一条路了，那就是直接把Xigncode在Java层剥离掉，不加载XigncodeClient，而是直接启动Unity进程。怎么做呢？我们来看反编译过后的smali代码：

![image-20250324164025961](https://s2.loli.net/2025/03/24/WOURHZ7fINDhMt5.png)

很明显这里对应了`XigncodeClientSystem.Callback`的实现，直接注释掉；

`OnHackDetected`整个方法都没用了，直接注释；

`onResume`、`onPause`、`onDestroy`，这三个方法里需要清除掉对`XigncodeClient`的调用：

![image-20250324164434161](https://s2.loli.net/2025/03/24/WXih4ZH6L95pKok.png)

然后，在OnCreate方法里，注释掉初始化`Xigncode`的逻辑，注意要保留向Unity发送初始化成功的那行代码，否则Unity会提示安全模块初始化失败：

```smali
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 3
    .annotation system Ldalvik/annotation/MethodParameters;
        accessFlags = {
            0x0
        }
        names = {
            "savedInstanceState"
        }
    .end annotation

    .line 31
    invoke-super {p0, p1}, Lcom/unity3d/player/UnityPlayerActivity;->onCreate(Landroid/os/Bundle;)V

    .line 33
    invoke-static {p0, p1}, Lcom/airisdk/sdkcall/AiriSDK;->onCreate(Landroid/app/Activity;Landroid/os/Bundle;)V

    const-string v0, "{\"succeeded\": true}"
    const-string v1, "XIGNCODE3Interface"
    const-string v2, "OnInitialized"

    invoke-static {v1, v2, v0}, Lcom/unity3d/player/UnityPlayer;->UnitySendMessage(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V

    return-void
.end method
```

OK，到这里就基本完成了xc的去除了，接着就是打包，签名，安装。这下终于不会弹反作弊弹窗，也不会闪退了。

不过这个解决方法毕竟是不完美的，甚至可以说没什么用的（因为直接没法登录官方服务器了），不过更进一步的办法也不是没有，可惜这里空白太小，笔者写不下（bushi