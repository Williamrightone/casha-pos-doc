# Git 分支策略

## 分支

本專案採簡化的 GitHub Flow，分支共為以下幾種：

* main branch：主要分支，隨時可上線的版本。禁止直接修改，僅能由 feature 分支合併，所有分支均需自 main 建立。
* dev branch：測試用分支，可依需求分為 sit、uat。
* feature branch：功能開發分支，自 main 建立，完成後先合併至 dev 測試，測試通過後再合併回 main。
* hotfix branch：緊急修復分支，自 main 建立，流程與 feature 相同（先合併 dev 測試，驗證後再回 main），僅在命名與優先順序上區別。

## 流程

1. checkout 專案後，確認自己位於 main。
2. 從 main 建立分支：
   * 功能開發：feature/[featurename]
   * 緊急修復：hotfix/[issuename]
3. 開發完成後 commit & push，並確保 SonarQube 覆蓋率達 80%，且無重大 issue。
4. 建立 PR，合併至 dev 進行 SIT/UAT 測試。
5. 若測試失敗，於同一分支修正並重複步驟 3–4。
6. 測試成功後，待上線指令，將已驗證完成的分支逐一合併至 main。
7. 於 main 打版號（tag），再執行 CD pipeline 部署。

## git script

```cmd
git checkout main
git checkout -b feature/my-feature

git add .
git commit -m "feat: implement my feature"
git push -u origin feature/my-feature
```
