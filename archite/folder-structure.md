# 專案目錄結構

Casha-Pos 的後端採用 DDD + Clean Architecture; packeage 則使用 maven-multi project.

由於專案採 Spring Cloud [微服務架構](./service-typology.md), 故服務分為 Backend For Frontend (BFF) 和 Business (Biz), 兩者各有相應的規則。

這裡先介紹 Class 的用途與規則, 依賴的方向, 再介紹共用 package 的管理, 最後呈現實際開發的專案資料夾結構。

## Class 規則

這裡僅介紹功能的用途與規定, 關於命名請參照 [命名規則](../spec/common/naming-rules.md)

**BFF Controller**
統一使用 http.post, 攜帶的 request & response 需繼承 BaseRq 和 BaseRs, 裡面包含一次性交易的





## 依賴的方向

**BFF**

Controller -> UseCase -> UseCaseImpl -> FeignClient

**Biz**

Controller -> UseCase -> UseCaseImpl -> Repository or Domain Service

## Common Package

專案的



## 結構


