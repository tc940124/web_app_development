# 流程圖文件 (Flowchart)

本文件根據需求 (PRD) 與系統架構 (Architecture)，規劃出「食譜收藏夾系統」的使用者操作邏輯動線，與核心功能的系統序列資料流方向。

---

## 1. 使用者流程圖 (User Flow)

描述「一般使用者」從進入網站到尋找食譜、並加入收藏的操作路徑與決策點。
*(此圖表使用 Mermaid 語法撰寫)*

```mermaid
flowchart LR
    Start([進入網站]) --> Home[網站首頁 - 食譜推薦列表]
    
    Home --> SearchPhase{想要尋找什麼？}
    SearchPhase --> |輸入關鍵字| KeywordSearch[關鍵字搜尋結果]
    SearchPhase --> |選取手邊食材| IngredientSearch[食材組合搜尋結果]
    SearchPhase --> |隨意看看| RecipeDetail[查看特定食譜內頁]
    
    KeywordSearch --> RecipeDetail
    IngredientSearch --> RecipeDetail
    
    RecipeDetail --> ActionFav{是否想收藏此食譜？}
    
    ActionFav --> |是| CheckLogin1{系統檢查是否已登入？}
    CheckLogin1 --> |未登入| Login[前往登入/註冊頁面]
    Login --> |登入成功| RecipeDetail
    CheckLogin1 --> |已登入| SuccessFav[成功加入收藏夾]
    
    Home --> NavProfile[點擊「我的收藏夾」]
    NavProfile --> CheckLogin2{系統檢查是否已登入？}
    CheckLogin2 --> |未登入| Login
    CheckLogin2 --> |已登入| MyFavList[進入個人收藏夾清單]
    MyFavList --> RecipeDetail
```

---

## 2. 系統序列圖 (Sequence Diagram)

此序列圖描述核心互動：「使用者點擊加入收藏」到「資料存入 SQLite」的完整系統內部交握流程。
*(此圖表使用 Mermaid 語法撰寫)*

```mermaid
sequenceDiagram
    actor User as 使用者
    participant Browser as 瀏覽器 (前端)
    participant Flask as Flask Route (Controller)
    participant Model as SQLAlchemy (Model)
    participant DB as SQLite 資料庫

    User->>Browser: 於食譜頁面點擊「收藏」按鈕
    Browser->>Flask: 發送 POST /recipe/{id}/favorite 請求
    
    Flask->>Flask: 驗證 Session 登入狀態
    
    alt 若使用者未登入
        Flask-->>Browser: 回傳 302 重導向 /auth/login
        Browser-->>User: 顯示登入畫面並提示需先登入
    else 若使用者已登入
        Flask->>Model: 呼叫查詢該使用者與該食譜的關聯
        Model->>DB: 轉換為 SQL SELECT
        DB-->>Model: 回傳查詢結果
        
        alt 發現已存在收藏紀錄
            Flask->>Model: 執行移除收藏邏輯
            Model->>DB: 轉換為 SQL DELETE
        else 尚未收藏
            Flask->>Model: 建立新的收藏紀錄 Model 物件
            Model->>DB: 轉換為 SQL INSERT INTO
        end
        
        DB-->>Model: 資料庫操作成功
        Model-->>Flask: 回傳狀態給 Controller
        Flask-->>Browser: 回傳 302 重導向回原食譜頁面 (帶入成功 Flash 訊息)
        Browser-->>User: 顯示「已成功收藏/取消收藏」
    end
```

---

## 3. 功能清單對照表

根據 PRD 定義的 5 項功能，規劃對應的 URL 路徑機制與 HTTP 方法。

| 功能區塊 | 操作動作 | 對應 URL 路徑 | HTTP 方法 | 備註說明 |
| :--- | :--- | :--- | :--- | :--- |
| **瀏覽與搜尋食譜** | 瀏覽首頁與推薦清單 | `/` | GET | 呈現全站食譜概覽 |
| | 關鍵字搜尋 | `/search` | GET | 帶有 `?q=關鍵字` 的查詢 |
| | 食材組合搜尋 | `/search/ingredients` | GET | 根據食材參數反查關聯食譜 |
| | 檢視食譜詳細資料 | `/recipe/<int:recipe_id>` | GET | 含圖片、材料表與步驟 |
| **儲存食譜** | 加入 / 取消收藏 | `/recipe/<int:recipe_id>/favorite` | POST | 點擊按鈕觸發的狀態切換，需登入 |
| | 查看我的收藏夾 | `/my/favorites` | GET | 呈現該會員的所有收藏，需登入 |
| **會員管理** | 註冊新帳號視圖 | `/auth/register` | GET | 顯示註冊表單 |
| | 提交註冊資料 | `/auth/register` | POST | 建立 User，密碼加密 |
| | 登入視圖 | `/auth/login` | GET | 顯示登入表單 |
| | 提交登入資料 | `/auth/login` | POST | 驗證密碼，派發 Session |
| | 登出 | `/auth/logout` | POST / GET | 清除 Session 跳回首頁 |
| **管理員功能** | 進入後台食譜列表 | `/admin/recipes` | GET | 需驗證管理員身分 |
| | 刪除不當食譜 | `/admin/recipe/<int:recipe_id>/delete` | POST | 將該食譜從資料庫連帶關聯中移除 |
