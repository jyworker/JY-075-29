# 修复解码器中 LanguagePreference 字段的 trim 问题

> **Workspace:** /home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-29

---

**User**

修复  MerchantInformationLanguageDecoder  解码  LanguagePreference  字段时，未对解析出的 value 执行  trim() ，导致带有前后空格的合法编码（如  " ZH " ）通过解码后在 validator 中格式校验失败的问题，应在 decoder 赋值前统一 trim，并同步更新对应测试用例。

**Assistant**

### Relevant Code Snippets
1. /home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-29/src/main/java/com/emv/qrcode/decoder/mpm/MerchantInformationLanguageDecoder.java:L1-L100
  — 找到了 MerchantInformationLanguageDecoder 类的定义，包括其 decode 方法和对 LanguagePreference 字段的处理逻辑。

2. /home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-29/src/main/java/com/emv/qrcode/model/mpm/MerchantInformationLanguage.java:L1-L100
  — 该文件包含了 MerchantInformationLanguage 类的定义，展示了 LanguagePreference 字段的结构和相关方法。
```

Now I understand the full flow. Let me also check the validator to understand the expected format:

Now I have a complete understanding of the issue. Let me update the todo and implement the fix:


Now let me implement the fix. The issue is in `MerchantInformationLanguageDecoder.decode()` at line 94, where decoded String values are passed to the consumer without trimming. I'll add trim() for String values:


Now let me update the test file to add a test case for the trim functionality. I'll add a test that verifies decoding values with surrounding spaces works correctly:



Now let me run the tests to verify the fix works correctly. First, let me check the build configuration:



The test is failing due to TLV length calculation issues with multi-byte Chinese characters combined with spaces. Let me simplify the test case to focus on the LanguagePreference field with spaces, using ASCII for other fields:




Now let me run the tests again:


I see the issue - the TLV length encoding is incorrect. Let me fix the test with proper TLV encoding:




Now let me run the tests again:




Let me check how the TLV iterator works to understand the parsing issue:






Let me check the template decoder to understand the full flow:








Let me run the test without quiet mode to see the full error details:










Let me check the field codes and write a simple debug to understand the parsing issue:













Let me create a simpler debug test first to understand the parsing issue, and then fix the test case:
















Let me simplify the test to focus only on the LanguagePreference field with spaces, which is the core issue:


















Let me fix the TLV length - 14 hex is incorrect. The inner content is 16 characters, so length should be 10 hex:



















Now let me run the tests again:
























Now I understand the issue! The length is stored as a **decimal** string, not hex. So "10" means 10 characters, not 16. Let me fix the test with the correct length:


























Now let me run the tests again:




























All tests pass! Now let me run the full test suite to ensure no regressions, and run diagnostics:





























All tests pass with no regressions, and no diagnostics issues. Let me finalize:










## 修复完成

### 问题分析
`MerchantInformationLanguageDecoder` 在解码 `LanguagePreference`（以及其他 String 类型字段）时，直接将 TLV 解析出的 value 传递给 setter，未对前后空格进行 trim 处理。导致类似 `" ZH "` 这样带空格的合法编码在解码后，被 validator 校验长度时失败（validator 要求 value 长度为 2，但 `" ZH "` 长度为 4）。

### 修改内容

**1. [MerchantInformationLanguageDecoder.java](file:///home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-29/src/main/java/com/emv/qrcode/decoder/mpm/MerchantInformationLanguageDecoder.java#L94-L100)**

在 `decode()` 方法中，对解码后的 String 类型值统一执行 `trim()` 后再赋值：

```java
final Object decodedValue = DecoderMpm.decode(value, clazz);

if (decodedValue instanceof String) {
  consumer.accept(result, ((String) decodedValue).trim());
} else {
  consumer.accept(result, decodedValue);
}
```

**2. [MerchantInformationLanguageDecoderTest.java](file:///home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-29/src/test/java/com/emv/qrcode/decoder/mpm/MerchantInformationLanguageDecoderTest.java#L98-L119)**

新增两个测试用例：
- `testSuccessDecodeWithSpacesInLanguagePreference`: 验证带前后空格的 `LanguagePreference` 解码后被正确 trim
- `testSuccessDecodeEncodeWithSpacesInLanguagePreference`: 验证解码后重新编码的结果不包含空格

### 验证结果
- ✅ 所有 6 个 `MerchantInformationLanguageDecoderTest` 测试通过
- ✅ 完整测试套件运行通过，无回归
- ✅ 无编译错误和 lint 警告

---

