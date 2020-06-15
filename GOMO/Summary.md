# QR Reader 2020-5-20

## 服务器端交互

服务端采用的是DES/ECB/PKCS5Padding的方式加密

iOS中CommonCrypt中DES加密默认是CBC模式的加密，而ECB/PKCS5Padding对应的模式是`kCCOptionPKCS7Padding | kCCOptionECBMode`

## 加密过程中中文字符问题

如果在加密过程中有中文字符，那么[UInt8]的长度则与不能是加密字符串的长度，因为中文字符会占3个字符长度。

所以需要对中文字符进行判断

## 加密过程实现

```Objective-C
+ (NSString *)desEncrypt:(NSString *) desString secret:(NSString *) secret
{
    size_t dataLength = [desString length];
    int chieseCount = 0;
    for (int i = 0; i < desString.length; i ++) {
        NSRange range = NSMakeRange(i, 1);
        NSString *subStr = [desString substringWithRange:range];
        const char *c = [subStr UTF8String];
        if (strlen(c) == 3) {
            chieseCount ++;
        }
    }
    dataLength = dataLength + chieseCount * 2;
    void const *textBytes = [desString UTF8String];
    uint8_t *bufferPtr = NULL;
    size_t bufferPtrSize = 0;
    size_t movedBytes = 0;
    bufferPtrSize = (dataLength + kCCBlockSizeDES) & ~(kCCBlockSizeDES - 1);
    bufferPtr = malloc(bufferPtrSize * sizeof(uint8_t));
    memset((void *)bufferPtr, 0x0, bufferPtrSize);
    CCCryptorStatus cryptStatus = CCCrypt(kCCEncrypt,
                                          kCCAlgorithmDES,
                                          kCCOptionPKCS7Padding|kCCOptionECBMode,
                                          [secret UTF8String],
                                          kCCKeySizeDES,
                                          NULL,
                                          textBytes,
                                          dataLength,
                                          (void *)bufferPtr,
                                          bufferPtrSize,
                                          &movedBytes);
    NSString *ciphretext;
    if (cryptStatus == kCCSuccess) {
    	NSData *base64Data = [NSData dataWithBytes:(const void *)bufferPtr length:(NSUInteger)movedBytes];
	NSData *desCryptData = [base64Data base64EncodeDataWithOptions: NSDataBase64EncodingWithCarriageReturn];
	ciphretext = [[NSString alloc] initWithData:desCryptData encoding: NSUTF8StringEncode];
    }
    return ciphretext;
}
```

## 其他注意事项

- httpBody中传数据不能直接传解密出来的data

- 对加密后数据进行Base64URLSafe后可能会改掉数据内容，导致解密失败

**步骤**

1. 将data转成base64字符串

2. 将base64字符用utf8转成data

3. 放到httpbody中

# QR Reader 2020-5-29

## OC Swift混编项目注意事项

Swift模块会自动生成OC代码中间文件

在 `Target -> Build Settings -> Objective-C Generated Interface Header Name` 中设置，默认是根据swift模块来命名， 默认`$(SWIFT_MODULE_NAME)-Swift.h`

# 2020-6-3

## iOS工程添加cpp文件以及opencv导入工程

- 在混编工程中导入cpp文件，需要在PCH文件上加上

```
#ifdef __OBJC__
...header import
#endif
```

Xcode能编译.c .m .mm .cpp等后缀的文件，而pch文件是上述几种后缀文件共用的，但是在编译.c .cpp时，初夏难于法和OC不兼容的情况，导致编译出错

- 导入opencv库

报错，将源码中的`NO`改为`NO_EXPOSURE_COMPENSATOR`或`NO_EXPOSURE_COMPENSATOR = 0`

`opencv2/highgui/cap_ios.h file not found`

cap_ios.h在3.2.0版本移动到videoio文件夹下面，更改路径


# 2020-6-12
### OC BOOL取值

NSDictionary 直接取 BOOL可能会出现错误。

```
NSDictionary dic = @{@"boolValue": @(NO)};
// po dic  
// {
//   boolValue = 0
// }
BOOL wrong = dic[@"boolValue"]; -> YES
NSNumber boolNum = dic[@"boolValue"];
BOOL right = [boolNum boolValue]; -> NO
```

# 2020-6-15
### 适配不同屏幕

ios 屏幕中逻辑分辨率大致分为 `320x480 iphone se` `375x667 iphone6/7/8` `414x736 iphone 6+/7+/8+` `375x812 iphonex/xs/xr/xsmax`

因为414 和 320屏幕宽度不同，设计大多用6/7/8屏幕尺寸设计，所以为了让界面看起来比例一样，可以按比例换算下开发单位points

以375位标准的换算

```
func sizeFit375(size: Float) {
	let temp = size / 375
	return temp * Screen.main.bounds.width
}
```
