# 基于Ionic，通过改写android启动View实现启动动画加载需求

* 在查阅相关cordova插件后了解到目前暂无能够支持app一打开就展示动画splashscreen效果的插件 ( 现有的cordova-plugin--splashscreen插件也只是通过一个静态图进行遮挡启动的闪屏现象 ) 
* 通过翻看...\platforms\android\src\com\picaex\app\MainActivity.java,了解到其实Ionic的android启动方式就是在View里面打开网页。</br>
* 为了实现app一打开就展示动画splashscreen效果，只能通过android原生进行修改启动流程

## 参考
1. [修改启动流程](http://students.ceid.upatras.gr/~besarat/JB/Blog/Entries/2015/3/13_cordova_how_to_create_animated_splashscreen_android.html)（注：使用后发现，loadUrl无发正常加载页面）
2. [基于（参考1），通过添加View](https://www.codeproject.com/Articles/113831/An-Advanced-Splash-Screen-for-Android-App)（注：需要结合参考1）

## 如何修改（PS：其实因该是从创建单文件，最后到设置View这个流程，如果没看参考直接看这块可能需要倒序看）
* 一个View情况下，先运行loading动画效果，再加载Ionic页面，会导致无法正常加载Ionic。所以新增一个View来解决这个问题：打开app -> 动画View -> 动画结束 -> 调用IonicView。

1. 找到...\platforms\android\AndroidManifest.xml，找到视图如：（这里就是android的启动Ionic视图）
```
......

<activity android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale" android:label="@string/activity_name" android:launchMode="singleTop" android:name="MainActivity" android:theme="@android:style/Theme.DeviceDefault.NoActionBar" android:windowSoftInputMode="adjustResize">
            <intent-filter android:label="@string/launcher_name">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
......

```
2. 在这基础上，新增一个视图，用于展示loading动画效果，但是2个视图我们需要优先级，这时候就要看`<category android:name="android.intent.category.LAUNCHER" />`，`android.intent.category.LAUNCHER` 表示优先启动显示，而`android.intent.category.DEFAULT` 表示暂不显示。（属于android范围，个人理解，需要更多信息的内容请查阅相关资料）。修改后如：

```
......
 <activity android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale" android:label="@string/activity_name" android:launchMode="singleTop" android:name="MainActivity" android:theme="@android:style/Theme.DeviceDefault.NoActionBar" android:windowSoftInputMode="adjustResize">
            <intent-filter android:label="@string/launcher_name">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
        <activity android:name="SplashScreen">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
......
```

3. 属性`android:name="<name>"`，表示对应的java文件（`...\platforms\android\src\com\(appId)\app\<name>.java`），根据新增视图的name（SplashScreen），在`...\platforms\android\src\com\(appId)\app\`下新增`SplashScreen.xml`文件，根据 [ 参考1 ]的内容进行代码填入:（动效视图完成动画后调取Ionic视图）
```
package  gr.your.pacjage;

import  gr.your.pacjage.R;
import org.apache.cordova.*;
import android.os.Bundle;


import android.view.View;

import android.graphics.PixelFormat;

import android.view.Window;

import android.view.animation.Animation;

import android.view.animation.AnimationUtils;

import android.widget.ImageView;

import android.widget.RelativeLayout;

import android.content.Intent;  
public class SplashScreen extends CordovaActivity {
	
       public void onAttachedToWindow() {

            super.onAttachedToWindow();

            Window window = getWindow();

            window.setFormat(PixelFormat.RGBA_8888);

        }



    @Override

    public void onCreate(Bundle savedInstanceState)

    {

        super.onCreate(savedInstanceState);


        setContentView(R.layout.splash_layout);

        AnimateNative();

        //loadUrl(launchUrl);

    }



    private void AnimateNative() {

        Animation anim = AnimationUtils.loadAnimation(this, R.anim.fade);

        anim.reset();

        RelativeLayout l=(RelativeLayout) findViewById(R.id.splash_lay);

        l.clearAnimation();

        l.startAnimation(anim);



        anim = AnimationUtils.loadAnimation(this, R.anim.translate);

        anim.reset();

        ImageView iv = (ImageView) findViewById(R.id.splash_img);

        iv.clearAnimation();

        iv.startAnimation(anim);

        anim.setFillAfter(true);

        anim.setAnimationListener(new Animation.AnimationListener(){

            @Override
            public void onAnimationStart(Animation arg0) {}           

            @Override
            public void onAnimationRepeat(Animation arg0) {}

            @Override
            public void onAnimationEnd(Animation arg0) {

               	finish();
    			// Run next activity
    			Intent intent = new Intent();
    			intent.setClass(SplashScreen.this, MainActivity.class);
    			startActivity(intent);
    			// stop();   
            }
        });
    }
}

```

4. 新增布局文件`...\platforms\android\res\layout\splash_layout.xml`（ps: 需要在`@drawable`用到splaschback.png和logo.png，用于测试随便内容图片就行）
```
<?xml version="1.0" encoding="utf-8"?>

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/splash_lay"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:layout_gravity="center"
    android:background="@drawable/splaschback"
    android:orientation="vertical">

    <ImageView

        android:id="@+id/splash_img"

        android:layout_width="wrap_content"

        android:layout_height="wrap_content"

        android:layout_alignParentBottom="true"

        android:layout_centerHorizontal="true"

        android:layout_marginBottom="22dp"

        android:src="@drawable/logo" /> 

</RelativeLayout>
```
5. 新增2个动画文件`...\platforms\android\res\anim\fade.xml`和`...\platforms\android\res\anim\translate.xml`
```
fade.xml

<?xml version="1.0" encoding="utf-8"?>
<alpha
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromAlpha="0.0"
    android:toAlpha="1.0"
    android:duration="3000" />


translate.xml


<?xml version="1.0" encoding="utf-8"?>

<set

    xmlns:android="http://schemas.android.com/apk/res/android">

    <translate


        android:fromXDelta="0%"

        android:toXDelta="2%"

        android:fromYDelta="100%"

        android:toYDelta="0%"

        android:duration="5000"

       />

</set>
```

6. 没了