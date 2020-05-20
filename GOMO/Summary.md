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
