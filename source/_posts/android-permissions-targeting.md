---
title: Android动态权限适配简介
author: lidu
date: 2017-12-25 16：13
tags: Android
---

# Android动态权限适配简介

## 前言

Android6.0系统早在2015年5月份发布，其中一个亮点就是增加了动态权限的功能。由于国内大部分用户无法第一时间接收到新系统的推送，且国内ROM厂商也需要一段时间去定制系统，因此适配新系统一直得不到开发者有效的重视。

时至今日，Android6.0系统已经面世两年多，国内诸多Android手机用户已经使用6.0+的操作系统。以百度外卖Android客户端为例，我们在百度统计平台上采集2017-10-30至2017-11-28期间的启动用户的Android操作系统分布的数据，可以看出使用Android6.0+系统的用户已经占比达到70%。为了提供给用户更好的用户体验和隐私功能，动态权限的适配是开发者绕不过去的一道坎。

![](https://ws1.sinaimg.cn/large/dc50da5fly1flxv5v44r4j20sg0lc42v.jpg)

## 基础要点

Android6.0系统是一道分水岭，在6.0系统之前，所有的系统权限都被一视同仁，开发者在AndroidManifest.xml中定义好所有APP需要的权限，然后在应用被安装时，用户只能接受所有权限并安装APP，或者拒绝安装。在应用运行时，所有预先在AndroidManifest.xml定义好的权限默认永久被授予，无需提前申请。

而在Android6.0及以后的系统中，系统权限被划分为三类：普通权限、危险权限和特殊权限。普通权限的流程和之前一样，不赘述。危险权限是指涉及用户隐私的部分权限（例如电话、存储、相机等），共有24个，由Google定义，[详细列表在此](https://developer.android.com/guide/topics/security/permissions.html?hl=zh-cn#defining)。危险权限不仅需要在AndroidManifest.xml中预先定义，而且在APP运行过程中需要每次都请求授权。这部分权限是我们动态权限适配的重点内容。特殊权限包含SYSTEM_ALERT_WINDOW和WRITE_SETTINGS，指的是设置悬浮窗和修改系统设置，不在此次讨论范畴。

Android6.0新增的另一个概念是权限组。权限组是针对24个危险权限进行的分组，目前共有9个权限组，分别为：

1. CALENDAR（日历）
2. CAMERA（相机）
3. CONTACTS（通讯录）
4. LOCATION（位置）
5. MICROPHONE（麦克风）
6. PHONE（电话信息）
7. SENSORS（传感器）
8. SMS（短信）
9. STORAGE（存储）

每个权限组包含一个或者更多危险权限，当某个权限组包含多个危险权限时，开发者申请某个危险权限被允许后，该权限组中的其他权限也一并被允许。

举例来说，STORAGE权限组包含READ_EXTERNAL_STORAGE和WRITE_EXTERNAL_STORAGE两个权限，如果我们在APP运行的时候需要读取手机存储内容，这个时候需要向系统申请READ_EXTERNAL_STORAGE权限，当用户允许后，如果APP后续需要读取或者写入存储，都无需再次申请权限，因为写入的权限与读取的权限被捆绑在一起，当其中任何一个权限被用户允许，系统将允许该权限组的所有权限。虽然有权限组的机制存在，但是还是建议动态申请所有需要的权限，而不是权限组中的某一个，因为Google后续可能更新权限组的分类，甚至更改权限组的这种机制。

## 对封装库的一些看法

作为开发者，我们非常关心当系统新增一项功能，其对开发者开放的API是否友好，动态权限的申请也是如此。我的想法是，官方的Framework层API设计是经过Google团队审慎思考后才对外发布的，所以其权威性是值得尊重的，虽然我们开发者经常吐槽某某API用起来一点也不好用，而且越来越懒的我们希望API尽可能简洁，最好一行代码就能搞定。在这个想法的驱使下，我们也希望将动态权限的适配封装成一个库，以提供更好的兼容性和更简洁的接口。

下面先总结Framework层API，然后介绍一下现有的比较成熟的开源库以及遇到的问题，最后分析我们的封装库的使用方法和设计思路。

## Framework层开放API

以下方法对应API Level为23及以上，因此在使用下列方法前需要做好版本判断。

### 1.检查某个权限是否被允许
```java
Context.checkSelfPermission(@NonNull String permission);
```
该方法用于检查某项危险权限是否被系统所允许，如授权允许，返回值为PackageManager#PERMISSION_GRANTED，如授权失败，返回PackageManager#PERMISSION_DENIED

注意：这个方法是Context类的抽象方法，具体实现在Context的子类中，因此如果某个Fragment对象想要检查权限，需要由对应Activity来查询。

### 2.向系统发送权限申请请求
```java
Activity.requestPermissions(@NonNull String[] permissions, int requestCode);

Fragment.requestPermissions(@NonNull String[] permissions, int requestCode);
```
但我们知道某些权限未被授权允许后，可以通过该方法去发送权限申请的请求。该方法在Activity或者Fragment类中都有，提供所需要申请的权限数组和唯一标识符Request Code即可发送请求。

权限的请求过程是一个跨进程的通信流程，通过Activity新建Intent，并在Intent中把权限数组（String[]）放入Extra中，启动系统权限弹窗，用户点击后再把结果回调到Activity中。

有意思的是，Frgament请求权限的流程是通过对应Activty去实现的，也就是说Activtiy管理着多个Fragment，如果其中有某个Fragment需要请求系统权限，那么Activity会为这个Frgament生成一个唯一标识并保存起来，然后走Activty的请求权限流程，最后在权限结果分发时，通过唯一标识符分发到对应Fragment的回调方法里。感兴趣的读者可以深入研究研究。

### 3.获取权限申请的结果
```java
Activity.onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
            @NonNull int[] grantResults);
            
Fragment.onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
            @NonNull int[] grantResults);
```

在我们发送完权限申请的请求后，系统会回调Activity/Fragment类的onRequestPermissionsResult()方法，用户点击允许或者拒绝等等权限结果都存储在grantResults数组里，数组中的值一一对应permissions数组中的权限，遍历String[]，然后在对应index中取grantResults的值，即可获得权限请求的所有结果。

### 4.检查某个权限被拒绝且不再允许
```java
Activtiy.shouldShowRequestPermissionRationale(@NonNull String permission);

Fragment.shouldShowRequestPermissionRationale(@NonNull String permission);
```

当系统权限弹窗弹出后，用户可能拒绝了某个权限，且勾选了不再允许，这个方法可以检查某个权限是否是这种情况。方法返回false则是上述情况。此时，比较好的处理方式是，应用APP弹出一个弹窗，解释一下权限的用途，并引导用户去手动打开该权限。

## 现有的成熟开源框架

在Github上用关键词permissions搜索，并筛选出Java代码，按Star数量排序：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fm139934lsj21l80yg0z9.jpg)

这里截取了排名前三的开源库，分别是PermissionsDispatcher、RxPermissions和easypermissions，三个开源库分别各有特点。

### [PermissionsDispatcher](https://github.com/permissions-dispatcher/PermissionsDispatcher)

排名第一的PermissionsDispatcher的特点是运用了注解，开发者可以用几个简洁的注解来解决动态权限的适配问题。

```java
@RuntimePermissions
public class MainActivity extends AppCompatActivity {

    @NeedsPermission(Manifest.permission.CAMERA)
    void showCamera() {
        getSupportFragmentManager().beginTransaction()
                .replace(R.id.sample_content_fragment, CameraPreviewFragment.newInstance())
                .addToBackStack("camera")
                .commitAllowingStateLoss();
    }

    @OnShowRationale(Manifest.permission.CAMERA)
    void showRationaleForCamera(final PermissionRequest request) {
        new AlertDialog.Builder(this)
            .setMessage(R.string.permission_camera_rationale)
            .setPositiveButton(R.string.button_allow, (dialog, button) -> request.proceed())
            .setNegativeButton(R.string.button_deny, (dialog, button) -> request.cancel())
            .show();
    }

    @OnPermissionDenied(Manifest.permission.CAMERA)
    void showDeniedForCamera() {
        Toast.makeText(this, R.string.permission_camera_denied, Toast.LENGTH_SHORT).show();
    }

    @OnNeverAskAgain(Manifest.permission.CAMERA)
    void showNeverAskForCamera() {
        Toast.makeText(this, R.string.permission_camera_neverask, Toast.LENGTH_SHORT).show();
    }
}

```
在Sample中，我们看到如果一个Activity需要用到动态权限，添加@RuntimePermissions注解即可，同时在真正用到危险权限的方法前加上@NeedsPermission注解，然后权限的申请流程就无需使用者去关心，权限申请的结果分发到对应的注解方法里，如@OnShowRationale、@OnPermissionDenied、 @OnNeverAskAgain等。

这种运用注解的实现方式对开发者而言写起来非常轻松友好，而且对于后续的维护者而言，可读性也非常强，因此注解方式的开源库大受欢迎。

### [RxPermissions](https://github.com/tbruyelle/RxPermissions)

这个库是结合了RxJava的适配动态权限的开源库，对Rx感兴趣的可以深入了解，本文暂不讨论。

### [easypermissions](https://github.com/googlesamples/easypermissions)

easypermissions是Goolge团队出品的开源库。Google团队权威的设计思想和优秀的代码风格一向值得我们开发者学习膜拜，本文将尽笔者全力去分析其设计思路。

#### 1. 使用简介

基本的使用方法可以参考项目中的[README.md](https://github.com/googlesamples/easypermissions/blob/master/README.md)

#### 2. 设计思想

以项目中所写的[Sample](https://github.com/googlesamples/easypermissions/tree/master/app/src/main/java/pub/devrel/easypermissions/sample)为例，我们来分析一下easypermissions动态申请权限的流程。

```java
    public void cameraTask() {
        if (hasCameraPermission()) {
            // Have permission, do the thing!
            Toast.makeText(this, "TODO: Camera things", Toast.LENGTH_LONG).show();
        } else {
            // Ask for one permission
            EasyPermissions.requestPermissions(
                    this,
                    getString(R.string.rationale_camera),
                    RC_CAMERA_PERM,
                    Manifest.permission.CAMERA);
        }
    }
```

进入到需要相机权限的方法内，首先判断了是否拥有相机权限，如果相机权限已经被授权允许，那么直接做相关事情，如果未被授权，调用EasyPermissions的静态方法requestPermissions(...)。

我们注意一下，这里传进去的第一个参数是this，在这个例子中就是Activity。Google考虑到兼容性和统一性的问题，其实也可以将Fragment和Support Library中的Fragment类当做this传进去，这样，对外的API其实有三个，分别是：

```java

    public static void requestPermissions(
            @NonNull Activity host, @NonNull String rationale,
            int requestCode, @NonNull String... perms) {
        requestPermissions(host, rationale, android.R.string.ok, android.R.string.cancel,
                requestCode, perms);
    }
    
    public static void requestPermissions(
            @NonNull Fragment host, @NonNull String rationale,
            int requestCode, @NonNull String... perms) {

        requestPermissions(host, rationale, android.R.string.ok, android.R.string.cancel,
                requestCode, perms);
    }
    
    public static void requestPermissions(
            @NonNull android.app.Fragment host, @NonNull String rationale,
            int requestCode, @NonNull String... perms) {

        requestPermissions(host, rationale, android.R.string.ok, android.R.string.cancel,
                requestCode, perms);
    }
```

这样对外的public方法名是统一的，但是实际传进来的host参数并不一致，而且后续真正发出权限请求的流程也不一致。巧妙的是，针对Activity、Frgament和Support Frgament这三种不一样的情况，在easypermissions内部使用了泛型的技巧来优雅地处理了兼容性和简洁API的特性。详细设计如下图所示：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fm3mzg3frbj20sa0sa7bw.jpg)
调用方（Activtiy、Frgament或者Support Fragment）请求权限的流程均通过EasyPermissions的静态方法来完成，但是有一个例外，系统权限申请的结果只能在调用方回调，也就是说系统只能回调Activtiy、Fragment或者Support Fragment的onRequestPermissions方法，因此在调用方的系统回调中必须把结果传递至EasyPermissions类中，让EasyPermissions来分发权限结果。注意该方法的最后一个参数其实就是最后EasyPermissions要分发到的Receiver，这里传了this，也就是调用方自己。

```java
	//Activtiy、Fragment、Support Fragment
    @Override
    public void onRequestPermissionsResult(int requestCode,
                                           @NonNull String[] permissions,
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        // EasyPermissions handles the request result.
        EasyPermissions.onRequestPermissionsResult(requestCode, permissions, grantResults, this);
    }
```

当把this传给EasyPermissions后，EasyPermissions会检查调用方是否实现了EasyPermissions.PermissionCallbacks接口，而在我们的case中，对应的调用方（Activtiy）实现了这个接口，所以结果最终还是可以到达调用方，只不过经历了一圈EasyPermissions类的转发。

![](https://ws1.sinaimg.cn/large/dc50da5fly1fm3mx1cse6j21300mi76e.jpg)

```java
	//EasyPermissions.java
    public static void onRequestPermissionsResult(int requestCode,
                                                  @NonNull String[] permissions,
                                                  @NonNull int[] grantResults,
                                                  @NonNull Object... receivers) {
        // Make a collection of granted and denied permissions from the request.
        List<String> granted = new ArrayList<>();
        List<String> denied = new ArrayList<>();
        for (int i = 0; i < permissions.length; i++) {
            String perm = permissions[i];
            if (grantResults[i] == PackageManager.PERMISSION_GRANTED) {
                granted.add(perm);
            } else {
                denied.add(perm);
            }
        }

        // iterate through all receivers
        for (Object object : receivers) {
            // Report granted permissions, if any.
            if (!granted.isEmpty()) {
                if (object instanceof PermissionCallbacks) {
                    ((PermissionCallbacks) object).onPermissionsGranted(requestCode, granted);
                }
            }

            // Report denied permissions, if any.
            if (!denied.isEmpty()) {
                if (object instanceof PermissionCallbacks) {
                    ((PermissionCallbacks) object).onPermissionsDenied(requestCode, denied);
                }
            }
        }
    }
```

#### 3. EasyPermissions库总结

Goolge自家封装的库优势在于对Activity、Frgament和Support Frgament等的兼容性上做得很好，同时利用泛型和代理机制，保证API的简洁性与易用性。

在将EasyPermissions应用于我们项目中时，发现有部分场景无法处理，具体表现为： 
 
1. 对于自定义UI控件，由于接收不到权限的系统回调（没有onRequestPermissionsResult方法），无法接收权限结果的通知
2. 对于Utils类，可能也需要获取危险权限（我们的WebSdk需要对H5提供通讯录的能力），在Utils类里同样也无法接收到权限的结果

为了解决这两个问题，我们封装了Permissions-helper库用于外卖的场景

## 封装库Permissions-helper

Permissions-helper库保留了EasyPermissions库的泛型和代理机制，能够对外提供较好的兼容性和一致性，同时我们将Permissions管理类（EasyPermissions.class）由单例模式改为实例模式，修改了Callback的注册时机。

### 1.使用方法

#### 1.1 新建PermissionsManager对象

在需要申请权限的页面（Activity、Fragment、自定义View）中新建默认的PermissionsManager对象

```java
private PermissionsManager mPermissionsManager = new PermissionsManager();

```

如果某个页面对权限的处理比较复杂，比如进入百度外卖APP的Splash页面后会对多组权限同时请求，且处理权限请求的结果种类非常繁多，逻辑复杂，可以考虑继承PermissionsManager类

```java
public class SplashPermissionsManager extends PermissionsManager {...}

private SplashPermissionsManager mPermissionsManager = new SplashPermissionsManager(this,
            new SplashPermissionsManager.SplashPermissionsListener() {
                @Override
                public void onSplashPermissionsDone() {
                    mAllPermissionGranted = true;
                    dispatchInSplash();
                    finish();
                }
            });
```

#### 1.2 申请对应权限

以申请单个相机权限为例，需要传入三个参数，第一个是Activity自身的引用，第二个是处理权限结果的Callback，第三个是需要请求的权限数组。

```java
mPermissionsManager.requestPermissions(MainActivity.this,
		new PermissionCallbacks() {
		@Override
		public void onPermissionsDenied(int requestCode, List<String> permissions) {
			if (isListEmpty(permissions)) {
				return;
			}
			if (PermissionsManager.isMarshmallowOrHigher()
				&& !mPermissionsManager
				.shouldShowRequestPermissionRationale(permissions.get(0))) {
				Toast.makeText(MainActivity.this, R.string.camera_always_denied,
					Toast.LENGTH_SHORT).show();
			} else {
				Toast.makeText(MainActivity.this, R.string.camera_permission_denied,
					Toast.LENGTH_SHORT).show();
			}
			
		}
		
		@Override
		public void onPermissionsGranted(int requestCode,
		List<String> permissions) {
			capturePhoto();
			
		}
	}, cameraPermissions);
			
```

#### 1.3 将权限申请结果交由PermissionsManager处理

在Activity/Fragment中的系统回调方法里把权限请求的结果交由对应PermissionsManager对象去处理

```java
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    mPermissionsManager.onRequestPermissionsResult(requestCode, permissions, grantResults);
}
```

### 2.设计思路

设计思路与EasyPermissions相似，参考了其泛型和代理的机制，同时为了处理外卖特有的场景，修改了两点重要的设计结构。

#### EasyPermissions的局限性

##### 1. 无法处理自定义UI控件需要请求权限的Case

设想如下的场景，当我们封装一个自定义的UI控件，比如说传图控件，因为涉及到相机权限，而且我们希望权限相关的适配工作也封装在该控件内部，这时候该如何操作呢？

前面已经提到过，Framework层的设计是只有Activity或者Fragment才可以发出权限申请请求并接收结果。因此我们最容易想到的思路就是，自定义View在被使用时拥有外部Activity的引用，这个时候我们通过这个Context引用去发出权限请求，然后在对应使用自定义View的Activtiy的系统回调里，再把权限结果交给自定义View去处理就好了。

这个思路本身没有问题，而且目前看来是唯一的解决办法。但是问题出现在EasyPermissions对这种思路根本不支持。下面将分为两类实现方式来详细阐述：

##### （1.1）让自定义控件实现EasyPermissions.PermissionCallbacks接口

让自定义控件实现该接口后，在Activity的系统回调方法中（onRequestPermissionsResult），传给EasyPermissions的Receiver不再是this，而是自定义控件，那么按理说权限申请的结果都会通知到自定义控件，然则不是。

```java
    private static void requestPermissions(
            @NonNull PermissionHelper helper, @NonNull String rationale,
            @StringRes int positiveButton, @StringRes int negativeButton,
            int requestCode, @NonNull String... perms) {

        // Check for permissions before dispatching the request
        if (hasPermissions(helper.getContext(), perms)) {
            notifyAlreadyHasPermissions(helper.getHost(), requestCode, perms);
            return;
        }

        // Request permissions
        helper.requestPermissions(rationale, positiveButton,
                negativeButton, requestCode, perms);
    }
    
    private static void notifyAlreadyHasPermissions(@NonNull Object object,
                                                    int requestCode,
                                                    @NonNull String[] perms) {
        int[] grantResults = new int[perms.length];
        for (int i = 0; i < perms.length; i++) {
            grantResults[i] = PackageManager.PERMISSION_GRANTED;
        }

        onRequestPermissionsResult(requestCode, perms, grantResults, object);
    }

```

原因在于EasyPermissions在发送权限申请之前，会先校验一遍权限，如果权限已经被允许了，就无须再次请求了。这里if(hasPermissions(...))就是一个校验过程，如果通过，则直接通知Activtiy权限已被允许。由于Activity并没有实现EasyPermissions.PermissionCallbacks接口，所以这个通知是无效的，也更加不可能到达自定义控件。

#####（1.2）让使用自定义控件的Activtiy实现EasyPermissions.PermissionCallbacks接口  

让Activtiy去实现EasyPermissions.PermissionCallbacks接口，可以解决（1.1）中的自定义控件接收不到权限结果的问题。这种实现方式需要自定义控件对外提供两个Public方法让Activity调用，从而把权限的结果传递给自定义控件。

但是这种实现方式实在并不优雅，仅仅是一个权限的申请结果，就经历了三层的传递（EasyPermissions、Activity、自定义控件），这个繁琐的过程会令使用者困惑不已。

##### 2. 无法处理Utils类需要申请权限的Case

在我们的WebSDK中，需要对H5提供一些能力，其中就包含通讯录这样的涉及到危险权限操作的能力，这时候如何适配动态权限又是一个头疼的问题。

具体问题可以描述为：在H5页面通过WebSDK协议想要获取手机通讯录信息，这时候会被WebSDKUtil类截获，在获取通讯录之前，需要对通讯录权限进行校验和申请，那这部分权限适配应该交给谁来完成呢？

幸运的是，我们的WebView是放在Frgament这个容器里，可以把相关的权限操作都放在Fragment中进行处理，然后把结果通知给这个Util类即可。这样的实现方式就跟上述（1.2）中一样，权限结果的传递过程繁琐，虽然粗暴有力却简陋无比。

在我们的封装库Permissions-helper里，对权限结果的传递进行了优化，如下图所示：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fm3ip059mcj214y0pagow.jpg)

#### Permissions-helper的改进之处

鉴于EasyPermissions库在非Activity/Fragment场景下的局限性，我们对其设计架构做出如下两点变化，以适应我们的场景需求：

##### 1. 核心类的实现方式由单例改为实例

EasyPermissions库中EasyPermissions这个最核心的类虽然不是单例类，但是其对外提供的方法全是static的，其思想就是一对多的页面管理方式，但是在APP运行期间，多个Activtiy或者Frgament同时请求权限的Case几乎不存在，因此我们的PermissionsManager的管理方式是一对一的实例模式。

这样做的优势在于：

 (1) 无须使用Map维护多个对象的状态，避免遍历Map的性能问题和对象引用的内存泄漏问题  

 (2) PermissionsManager是默认的实现类，使用者可以继承该类，去实现更加复杂的权限处理逻辑（比如进入APP后同时请求三组权限，对结果的判断逻辑很复杂）
 
 ![](https://ws1.sinaimg.cn/large/dc50da5fly1fm3lnzqbigj219u0m0whf.jpg)
  
##### 2. Callback注册时机提前

在EasyPermissions中，想要接收到权限结果的通知，必须实现EasyPermissions.PermissionCallbacks接口，但是该接口的注册是在Activity/Frgament的onRequestPermissionsResult的回调中，这个时机有点晚。我们在Pemissions-helper中将这个Callback的注册时机提前至申请权限时，而非权限结果的回调中，如下所示：

```java
    public void requestPermissions(Activity host, PermissionCallbacks callbacks,
                                   String... permissions) {
        mCallback = callbacks;
        mDelegate = PermissionsDelegate.newInstance(host);
                /*
                 * 如果已经拥有权限，通过Callback通知，并返回，不再请求权限
				 */
        if (hasPermissions(mDelegate.getContext(), permissions)) {
            notifyAlreadyHasPermissions(0, permissions);
            return;
        }
        mDelegate.directRequestPermissions(permissions);

    }
```

#### Permissions-helper全景图

![](https://ws1.sinaimg.cn/large/dc50da5fly1fm3ookt5vfj20ru0wk7bz.jpg)

## 参考内容

1. https://mtj.baidu.com/web/terminal?appId=53918

2. https://developer.android.com/training/permissions/requesting.html?hl=zh-cn#perm-request  

3. https://developer.android.com/guide/topics/security/permissions.html?hl=zh-cn#defining  

4. https://github.com/googlesamples/easypermissions

5. https://github.com/permissions-dispatcher/PermissionsDispatcher

6. https://github.com/search?l=Java&o=desc&q=permissions&s=stars&type=Repositories&utf8=%E2%9C%93

