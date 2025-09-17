# 異常處置

本系統採用分層的異常處理架構，主要包含以下組件：

* AbstractServiceException: 基礎自定義異常類
* BaseErrorType: 錯誤類型接口
* 各服務特定的異常類 (如 AuthServiceException)
* 全局異常處理器 (BffExceptionHandler)

這種設計提供了統一的錯誤處理機制，同時允許各服務定義自己的錯誤類型。

```java
public class AbstractServiceException extends RuntimeException {
    private final String errorCode;
    private final IErrorLevel errorLevel;
    private final ModelType modelType;
    private final String memo;

    public AbstractServiceException(BaseErrorType errorType, String exceptionMsg) {
        super(exceptionMsg);
        this.errorCode = errorType.getCustomErrorCode();
        this.errorLevel = errorType.getIErrorLevel();
        this.modelType = errorType.getModelType();
        this.memo = errorType.getMemo();
    }
    // Getters...
}
```

## 核心屬性說明

|屬性|類型|說明|
| -- | -- | -- |
|errorCode|String|錯誤代碼，用於識別特定錯誤|
|errorLevel|IErrorLevel|錯誤嚴重程度 (如 LOW, HIGH)|
|modelType|ModelType|模組類型標識 (如 AU 表示認證模組)|
|memo|String|錯誤的簡要描述|

## 服務特定異常實現

以認證服務為例，實現自定義異常：

```java
public class AuthServiceException extends AbstractServiceException {
    
    public enum AuthServiceErrorType implements BaseErrorType {
        USER_NOT_FOUND("00001", IErrorLevel.LOW, ModelType.AU, "查無用戶"),
        LOGIN_FAIL_OVER_THREE_TIME("00002", IErrorLevel.LOW, ModelType.AU, "登入失敗超過3次"),
        ACCOUNT_PASSWD_NOT_MATCH("00003", IErrorLevel.LOW, ModelType.AU, "帳號密碼不匹配"),
        ACCOUNT_BANNED("00004", IErrorLevel.LOW, ModelType.AU, "用戶帳號遭禁止"),
        ACCOUNT_ROLE_INACTIVE("00005", IErrorLevel.LOW, ModelType.AU, "用戶尚未綁定權限"),
        ACCOUNT_SESSION_SERIALIZE_FAILED("00006", IErrorLevel.HIGH, ModelType.AU, "用戶 Session 序列化失敗");
        
        // 構造方法和接口實現...
    }
    
    public AuthServiceException(BaseErrorType errorType) {
        super(errorType, errorType.getMemo());
    }
}
```

錯誤類型設計優點

1. 集中管理：所有錯誤類型在一個枚舉中定義
2. 標準化：統一包含錯誤碼、等級、模組和描述
3. 可擴展：新增錯誤只需添加新的枚舉值

## 異常處理實例

在服務方法中使用自定義異常：

```java
@Override
public SaasLoginRs loginByUserNameAndPwd(SaasLoginRq rq) {
    // 查詢用戶
    CashaUser user = cashaUserRepository.findByUserAccount(rq.getUserAccount())
            .map(CashaUser::fromEntity)
            .orElseThrow(() -> new AuthServiceException(AuthServiceException.AuthServiceErrorType.USER_NOT_FOUND));

    // 檢查登入失敗次數
    if (isAccountLoginFailOverThreeTimes(user.getUsername())) {
        throw new AuthServiceException(AuthServiceException.AuthServiceErrorType.LOGIN_FAIL_OVER_THREE_TIME);
    }

    // 驗證密碼
    if (!user.isCredentialsValid(rq.getPasswd(), passwordEncoder)) {
        incrementLoginFailCount(rq.getUserAccount());
        throw new AuthServiceException(AuthServiceException.AuthServiceErrorType.ACCOUNT_PASSWD_NOT_MATCH);
    }

    // 檢查帳號狀態
    if (!user.isActive()) {
        throw new AuthServiceException(AuthServiceException.AuthServiceErrorType.ACCOUNT_BANNED);
    }

    // 登入成功處理
    clearLoginFailCount(rq.getUserAccount());
    return loginSessionBuilderService.buildLoginSession(user);
}
```

## 全局異常處理器

```java
@Slf4j
@ControllerAdvice
public class BffExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse> handleMethodArgumentNotValidException(
            MethodArgumentNotValidException error) {
        // 處理參數驗證異常...
    }

    @ExceptionHandler(AbstractServiceException.class)
    public ResponseEntity<ApiResponse> handleCustomServiceException(AbstractServiceException error) {
        // 可根據需要將 HIGH 級錯誤存入DB或發送到MQ
        // errorLogPublisher.publishErrorLogEvent(error);
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ApiResponse(
                    error.getModelType() + error.getErrorCode(), 
                    error.getMessage(), 
                    null));
    }
}
```

## 全局處理器功能

1. 統一響應格式：所有異常轉換為標準的 ApiResponse
2. 錯誤分類處理：不同類型異常可採用不同處理邏輯
3. 錯誤日誌記錄：可集中處理錯誤日誌記錄和監控

> 推薦下一篇: [Unit Test](/spec/common/unit-test.md)