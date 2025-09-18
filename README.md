# Casha Pos

**Casha POS** 是一套專為餐飲業 (F&B) 打造的系統，採用 **Spring Boot 3.4.8** 與 **Spring Cloud 2024** 建構後端微服務，前端則以 **Vue 3 + 組合式 API** 搭配 **Element Plus** 開發。系統涵蓋後台管理、POS 點餐、KDS 廚房顯示與消費者自助點餐，並整合多種中介軟體與觀測工具，支援高併發交易與全鏈路監控。

![cover](/asset/casha-cover.png)

## 專案架構

詳見 [Casha POS 專案拓樸](/archite/service-typology.md) 和 [Infrastructure](/archite/folder-structure.md)

![preview](/asset/casha-typo.png)

![preview](/asset/casha-infra.png)

## 線上測試

共分為四個操作平台:

* 後台網站: https://casha.williamrightone.com
* Pos 點餐系統(尚未開放): https://casha-pos.williamrightone.com
* 用戶點餐系統: https://casha-order.williamrightone.com/m/ou3u4zzulxfwhmit
* KDS(尚未開放): https://casha-kds.williamrightone.com

提供平台管理員維護, 餐廳與分店管理員配置, 員工與用戶點餐的功能。

## 關於細節

還在整理中...

整體採 Vue3 + Spring Cloud 開發, 建議閱讀順序

1. [專案架構與拓樸](/archite/service-typology.md)
2. [專案目錄結構](/archite/folder-structure.md)
3. [開發流程](/spec/development.md)
4. [RBAC](/spec/common/rbac.md)
5. [菜單設置與快取機制](/spec/usecase/menu.md)
6. [點餐與付款 API](/spec/usecase/order.md)
7. [監控與壓力測試](/devops/ptom.md)