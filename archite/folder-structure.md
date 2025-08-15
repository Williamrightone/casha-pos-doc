# 專案目錄結構

Casha-Pos 的後端採用 DDD + Clean Architecture; packeage 則使用 maven-multi project.

由於專案採 Spring Cloud [微服務架構](./service-typology.md), 故服務分為 Backend For Frontend (BFF) 和 Business (Biz), 兩者各有相應的規則。

這裡先介紹 Class 的用途與規則, 依賴的方向, 再介紹共用 package 的管理, 最後呈現實際開發的專案資料夾結構。

## Class 規則

這裡僅介紹功能的用途與規定, 關於命名請參照 [命名規則](../spec/common/naming-rules.md)

**BFF Controller**
統一使用 http.post, 攜帶的 request & response 需繼承 BaseRq 和 BaseRs, 裡面包含一次性交易的 id 等。

返回的 response 統一由 [ApiResponse](../spec/common/apiResponse.md) 包裝。

**BFF UseCase**
定義該交易行為的介面。

**BFF UseCaseImpl**
實作交易行為, 內部可由多個 Domain Service, Feign 組合而成實作流程 pipeline, 若有必要於此實作 Sage Pattern 應付交易問題。

**BFF Domain Service**
Domain Service 只專注於 Bff 層與邏輯相關的方法。

**BFF Feign**
與 Biz 溝通之 api, 一個 Biz 可以有多個 Feign, 與 UseCase 相關, 以 contextId 管理。

```java
@FeignClient(name = "auth-service", contextId = "functionMenuClient")
public interface AuthFunctionMenuClient {

    @PostMapping("/auth/function")
    ApiResponse<FunctionQueryRs> queryFunctionMenus();

    @PostMapping("/auth/function/create")
    ApiResponse<CreateFunctionMenuRs> createFunctionMenu(@RequestBody CreateFunctionMenuRq rq);
}

@FeignClient(name = "auth-service", contextId = "permissionClient")
public interface AuthPermissionClient {

    @PostMapping("/auth/permission/findAll")
    ApiResponse<FindAllPermissionRs> findAllPermissions();

    @PostMapping("/auth/permission/create")
    ApiResponse<Void> createPermission(@RequestBody CreatePermissionRq rq);
}
```

**Biz Controller**




## 依賴的方向

**BFF**

Controller -> UseCase -> UseCaseImpl -> FeignClient

**Biz**

Controller -> UseCase -> UseCaseImpl -> Repository or Domain Service

## Common Package

專案的

**SHARED-COMMON**

**BFF-COMMON**

**BIZ-COMMON**

**Service-Contract**

每一個 biz-service 對應一個 contract, 如 auth-service 對應 auth-contract。

內容包含 AuthFeignClient、LoginRequest、UserInfoDTO 等，僅與 auth 服務相關。
bff 依賴它來調用 auth 服務。


## 結構


