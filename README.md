## iOS FaceID & TouchID

##### API文档 [System/LocalAuthentication](https://developer.apple.com/documentation/localauthentication?language=objc)
#### FaceID和TouchID本身基本逻辑很简单（当然需要置入一些安全相关的第三方SDK的情况下除外），先来介绍几个关键参数值和一些主意事项，然后直接贴代码就OK

###### LAPolicyDeviceOwnerAuthenticationWithBiometrics 和 LAPolicyDeviceOwnerAuthentication
- LAPolicyDeviceOwnerAuthenticationWithBiometrics
    - iOS 8.0+

```
// 设备所有者使用生物识别方法进行认证。TouchID验证是必须的，如果TouchID不可用或者没有注册，则策略评估将会失败。如果TouchID是锁被锁定，则需要输入密码作为解锁TouchID的第一步。
// TouchID对话框包括一个取消按钮，默认标题为“取消”，可以使用“localizedCancelTitle”属性去修改。有一个fallback按钮默认标题为“输入密码”，可以通过“localizedFallbackTitle”属性去修改。
// fallback按钮最初是隐藏的，并且在首次touchID尝试失败之后显示。
// 点击取消按钮或者输入密码按钮后会导致 evaluatePolicy: 方法调用失败，返回一个不同的错误代码。
// 5次验证失败，生物识别将被锁定。之后，用户必须输入密码才可以解锁验证。
```

- LAPolicyDeviceOwnerAuthentication
    - iOS 9.0+

```
// 设备所有者使用生物识别后者设备密码进行验证。
// TouchID或者密码验证是必须的，如果TouchID可用，注册并且未被锁定，首先使用TouchID验证。否则直接要求输入设备密码。
// 如果密码未被启用，则策略评估失败。
// Touch ID身份验证对话框的行为与LAPolicyDeviceOwnerAuthenticationWithBiometrics使用的行为相似。当点击输入密码按钮的时候，切换验证方法，并允许用户输入设备密码。
// 密码锁定会在6次失败尝试之后被锁定，并且逐步增加退避延迟。
```

###### iOS11新增属性 LABiometryType biometryType
- iOS 11.0+
- 这里需要注意的是请仔细阅读 biometryType 属性的注释。中文意思为只有当canEvaluatePolicy:生物识别策略成功之后才会去设置这个属性的值。简单来说意思就是 biometryType 这个属性的值只有在你调用canEvaluatePolicy:方法之后并且返回是YES没有错误的情况下才会设置，才会有值。在调用 canEvaluatePolicy: 方法前，或者调用后但是有Error的情况下，该属性均无任何有意义的值，验证之后实际为空。

```
typedef NS_ENUM(NSInteger, LABiometryType)
{
    /// The device does not support biometry.
    LABiometryNone,
    
    /// The device supports Touch ID.
    LABiometryTypeTouchID,
    
    /// The device supports Face ID.
    LABiometryTypeFaceID,
} NS_ENUM_AVAILABLE(NA, 11_0) __WATCHOS_UNAVAILABLE __TVOS_UNAVAILABLE;

/// Indicates the type of the biometry supported by the device.
///
/// @discussion  This property is set only when canEvaluatePolicy succeeds for a biometric policy.
///              The default value is LABiometryNone.
@property (nonatomic, readonly) LABiometryType biometryType NS_AVAILABLE(NA, 11_0) __WATCHOS_UNAVAILABLE __TVOS_UNAVAILABLE;
```





#### 代码附上
```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    // 检测设备是否支持TouchID或者FaceID
    if (@available(iOS 8.0, *)) {
        self.LAContent = [[LAContext alloc] init];
        
        NSError *authError = nil;
        BOOL isCanEvaluatePolicy = [self.LAContent canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&authError];
        
        if (authError) {
            NSLog(@"检测设备是否支持TouchID或者FaceID失败！\n error : == %@",authError.localizedDescription);
            [self showAlertView:[NSString stringWithFormat:@"检测设备是否支持TouchID或者FaceID失败。\n errorCode : %ld\n errorMsg : %@",(long)authError.code, authError.localizedDescription]];
        } else {
            if (isCanEvaluatePolicy) {
                // 判断设备支持TouchID还是FaceID
                if (@available(iOS 11.0, *)) {
                    switch (self.LAContent.biometryType) {
                        case LABiometryNone:
                        {
                            [self justSupportBiometricsType:0];
                        }
                            break;
                        case LABiometryTypeTouchID:
                        {
                            [self justSupportBiometricsType:1];
                        }
                            break;
                        case LABiometryTypeFaceID:
                        {
                            [self justSupportBiometricsType:2];
                        }
                            break;
                        default:
                            break;
                    }
                } else {
                    // Fallback on earlier versions
                    NSLog(@"iOS 11之前不需要判断 biometryType");
                    // 因为iPhoneX起始系统版本都已经是iOS11.0，所以iOS11.0系统版本下不需要再去判断是否支持faceID，直接走支持TouchID逻辑即可。
                    [self justSupportBiometricsType:1];
                }
                
            } else {
                [self justSupportBiometricsType:0];
            }
        }
        
    } else {
        // Fallback on earlier versions
        [self justSupportBiometricsType:0];
    }
}

#pragma mark -
#pragma mark -------------------- private actions --------------------
// 开始验证按钮点击事件
- (IBAction)didClickBtnCheck:(id)sender {
    NSString *myLocalizedReasonString = @"验证";
    
    [self.LAContent evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics localizedReason:myLocalizedReasonString reply:^(BOOL success, NSError * _Nullable error) {
        if (success) {
            NSLog(@"身份验证成功！");
            [self showAlertView:@"验证成功"];
        } else {
            // 做特定的错误判断处理逻辑。
            NSLog(@"身份验证失败！ \nerrorCode : %ld, errorMsg : %@",(long)error.code, error.localizedDescription);
            // error 参考 LAError.h
            [self showAlertView:[NSString stringWithFormat:@"身份验证失败！\nerrCode : %ld\nerrorMsg : %@",(long)error.code, error.localizedDescription]];
        }
    }];
}

#pragma mark -
#pragma mark -------------------- common methods --------------------
// 判断生物识别类型，更新UI
- (void)justSupportBiometricsType:(NSInteger)biometryType
{
    switch (biometryType) {
        case 0:
        {
            NSLog(@"该设备支持不支持FaceID和TouchID");
            self.lblMsg.text = @"该设备支持不支持FaceID和TouchID";
            self.lblMsg.textColor = [UIColor redColor];
            self.btnCheck.enabled = NO;
        }
            break;
        case 1:
        {
            NSLog(@"该设备支持TouchID");
            self.lblMsg.text = @"该设备支持Touch ID";
            [self.btnCheck setTitle:@"点击开始验证Touch ID" forState:UIControlStateNormal];
            self.btnCheck.enabled = YES;
        }
            break;
        case 2:
        {
            NSLog(@"该设备支持Face ID");
            self.lblMsg.text = @"该设备支持Face ID";
            [self.btnCheck setTitle:@"点击开始验证Face ID" forState:UIControlStateNormal];
            self.btnCheck.enabled = YES;
        }
            break;
        default:
            break;
    }
}

// 弹框
- (void)showAlertView:(NSString *)msg
{
    NSLog(@"%@",msg);
    if (@available(iOS 8.0, *)) {
        UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleCancel handler:nil];
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:nil message:msg preferredStyle:UIAlertControllerStyleAlert];
        [alertController addAction:okAction];
        [self presentViewController:alertController animated:YES completion:nil];
    } else {
        // Fallback on earlier versions
        UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:nil message:msg delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil, nil];
        [alertView show];
    }
}
```

##### LAError

```
typedef NS_ENUM(NSInteger, LAError)
{
    //身份验证不成功，因为用户无法提供有效的凭据。
    LAErrorAuthenticationFailed = kLAErrorAuthenticationFailed,

    //认证被用户取消(例如了取消按钮)。
    LAErrorUserCancel = kLAErrorUserCancel,

    //认证被取消了,因为用户利用回退按钮(输入密码)。
    LAErrorUserFallback = kLAErrorUserFallback,

    //身份验证被系统取消了(如另一个应用程序去前台)。
    LAErrorSystemCancel = kLAErrorSystemCancel,

    //身份验证无法启动,因为设备没有设置密码。
    LAErrorPasscodeNotSet = kLAErrorPasscodeNotSet,

    //身份验证无法启动,因为触摸ID不可用在设备上。
    LAErrorTouchIDNotAvailable NS_ENUM_DEPRECATED(10_10, 10_13, 8_0, 11_0, "use LAErrorBiometryNotAvailable") = kLAErrorTouchIDNotAvailable,

    //身份验证无法启动,因为没有登记的手指触摸ID。
    LAErrorTouchIDNotEnrolled NS_ENUM_DEPRECATED(10_10, 10_13, 8_0, 11_0, "use LAErrorBiometryNotEnrolled") = kLAErrorTouchIDNotEnrolled,

    //验证不成功,因为有太多的失败的触摸ID尝试和触///摸现在ID是锁着的。
    //解锁TouchID必须要使用密码，例如调用LAPolicyDeviceOwnerAuthenti//cationWithBiometrics的时候密码是必要条件。
    //身份验证不成功，因为有太多失败的触摸ID尝试和触摸ID现在被锁定。
    LAErrorTouchIDLockout NS_ENUM_DEPRECATED(10_11, 10_13, 9_0, 11_0, "use LAErrorBiometryLockout")
    __WATCHOS_DEPRECATED(3.0, 4.0, "use LAErrorBiometryLockout") __TVOS_DEPRECATED(10.0, 11.0, "use LAErrorBiometryLockout") = kLAErrorTouchIDLockout,

    //应用程序取消了身份验证（例如在进行身份验证时调用了无效）。
    LAErrorAppCancel NS_ENUM_AVAILABLE(10_11, 9_0) = kLAErrorAppCancel,

    //LAContext传递给这个调用之前已经失效。
    LAErrorInvalidContext NS_ENUM_AVAILABLE(10_11, 9_0) = kLAErrorInvalidContext,

    //身份验证无法启动,因为生物识别验证在当前这个设备上不可用。
    LAErrorBiometryNotAvailable NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryNotAvailable,

    //身份验证无法启动，因为生物识别没有录入信息。
    LAErrorBiometryNotEnrolled NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryNotEnrolled,

    //身份验证不成功，因为太多次的验证失败并且生物识别验证是锁定状态。此时，必须输入密码才能解锁。例如LAPolicyDeviceOwnerAuthenticationWithBiometrics时候将密码作为先决条件。
    LAErrorBiometryLockout NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryLockout,

    //身份验证失败。因为这需要显示UI已禁止使用interactionNotAllowed属性。  据说是beta版本
    LAErrorNotInteractive API_AVAILABLE(macos(10.10), ios(8.0), watchos(3.0), tvos(10.0)) = kLAErrorNotInteractive,
} NS_ENUM_AVAILABLE(10_10, 8_0) __WATCHOS_AVAILABLE(3.0) __TVOS_AVAILABLE(10.0);

```
---
### Info
- Blog：https://echozuo.github.io
- jianshu：https://www.jianshu.com/u/3390ce71084e
- CSDN：https://blog.csdn.net/zuoqianheng
- Email: zuoqianheng@foxmail.com
- Telegram：@echozuo
