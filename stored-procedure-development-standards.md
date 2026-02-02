# Stored Procedure 開發規範

> [!IMPORTANT]
> 本規範為強制性開發標準，所有開發人員在撰寫 Stored Procedure 時必須嚴格遵守。違反規範可能導致嚴重的效能問題、安全漏洞或系統穩定性問題。

## 目錄
- [命名規範](#命名規範)
- [註解標準](#註解標準)
- [效能優化規範](#效能優化規範)
- [資料操作規範](#資料操作規範)
- [禁止使用的做法](#禁止使用的做法)
- [錯誤處理規範](#錯誤處理規範)
- [安全性規範](#安全性規範)
- [程式碼品質規範](#程式碼品質規範)
- [程式碼審查檢查清單](#程式碼審查檢查清單)

---

## 命名規範

### 1. Stored Procedure 命名
**規範：**
- 使用 `usp_` (User Stored Procedure) 作為前綴
- 格式：`usp_[模組名稱]_[動作]_[對象]`
- 使用 Pascal Case（每個單字首字母大寫）
- 名稱必須清楚描述功能
- 禁止重複命名或功能重複的 SP

**標準化動作動詞：**
| 動作 | 說明 | 範例 |
|------|------|------|
| `Get` / `Select` / `Retrieve` | 查詢資料 | usp_Order_Get_ByCustomer |
| `Add` / `Insert` / `Create` | 新增資料 | usp_Order_Add_WithDetails |
| `Update` / `Modify` / `Edit` | 更新資料 | usp_Order_Update_Status |
| `Delete` / `Remove` | 刪除資料 | usp_Order_Delete_ByID |
| `Upsert` / `Merge` | 插入或更新 | usp_Product_Upsert_Inventory |
| `Process` / `Execute` | 處理邏輯 | usp_Invoice_Process_Monthly |

**正確範例：**
```sql
usp_Order_Add_CustomerOrder      -- ✅ 使用 Add 表示新增
usp_User_Update_Profile          -- ✅ 使用 Update 表示更新
usp_Report_Get_MonthlySales      -- ✅ 使用 Get 表示查詢
usp_Order_Delete_ByID            -- ✅ 使用 Delete 表示刪除
```

**錯誤範例：**
```sql
sp_InsertOrder                   -- ❌ 不使用 sp_ 前綴（保留給系統）
proc_get_data                    -- ❌ 命名不清楚
usp_DoStuff                      -- ❌ 命名無意義
usp_Order_Add_New                -- ❌ 已有相同功能的 usp_Order_Add_CustomerOrder
```

**原因：**
- `sp_` 前綴保留給系統 stored procedure，使用會導致效能問題（SQL Server 會先在 master 資料庫搜尋）
- 清楚的命名有助於維護和理解程式碼用途
- 標準化動詞避免混淆（如 Insert vs Add）
- 防止功能重複導致維護困難

### 2. Schema 前綴（強制）

**規範：**
❗ 所有資料表、視圖、函數引用都必須明確指定 Schema（通常為 `dbo.`）

**正確範例：**
```sql
-- ✅ 明確指定 Schema
SELECT CustomerID, CustomerName 
FROM dbo.Customers 
WHERE CustomerID = @CustomerID;

-- ✅ JOIN 也要明確指定
SELECT o.OrderID, c.CustomerName
FROM dbo.Orders o
INNER JOIN dbo.Customers c ON o.CustomerID = c.CustomerID;

-- ✅ INSERT/UPDATE/DELETE 也要指定
INSERT INTO dbo.Orders (CustomerID, OrderDate) 
VALUES (@CustomerID, GETDATE());
```

**錯誤範例：**
```sql
-- ❌ 缺少 Schema 前綴
SELECT CustomerID FROM Customers;
UPDATE Orders SET Status = 'Complete';
```

**原因：**
- 避免物件名稱解析延遲（SQL Server 需要查找預設 Schema）
- 提升查詢效能（特別是頻繁呼叫的 SP）
- 避免當使用者預設 Schema 不同時的錯誤
- 提高程式碼可讀性和明確性

### 3. 參數命名
**規範：**
- 使用 `@` 前綴（SQL Server 強制）
- 格式：`@[描述性名稱]`
- 使用 Pascal Case
- 輸出參數使用 `@Output` 後綴
- 參數資料型別和長度必須與資料表欄位一致

**正確範例：**
```sql
@CustomerID INT,                                    -- 與 dbo.Customers.CustomerID 型別一致
@OrderDate DATETIME,                                -- 與 dbo.Orders.OrderDate 型別一致
@CustomerName NVARCHAR(100),                        -- 與 dbo.Customers.CustomerName 長度一致
@TotalAmountOutput DECIMAL(18,2) OUTPUT             -- 輸出參數使用 Output 後綴
```

**錯誤範例：**
```sql
@CustomerName NVARCHAR(50)      -- ❌ 資料表欄位是 NVARCHAR(100)，長度不一致
@OrderDate DATE                 -- ❌ 資料表欄位是 DATETIME，型別不一致
```

**原因：**
- 參數型別不一致會導致隱式轉換，嚴重影響效能和索引使用
- 參數長度不足可能導致資料截斷錯誤
- 保持一致性提高程式碼可維護性

---

## 註解標準

### 1. Stored Procedure 標頭註解（強制）

**規範：**
每個 Stored Procedure 必須包含以下標頭資訊：

```sql
/*******************************************************************************
* Stored Procedure: usp_Order_Insert_CustomerOrder
* 說明: 新增客戶訂單及訂單明細
* 
* 作者: [開發人員姓名]
* 建立日期: YYYY-MM-DD
* 最後修改日期: YYYY-MM-DD
* 修改者: [修改人員姓名]
* 
* 參數說明:
*   @CustomerID INT - 客戶ID
*   @OrderDate DATETIME - 訂單日期
*   @OrderItems XML - 訂單項目（XML格式）
*   @OrderIDOutput INT OUTPUT - 返回新建訂單ID
* 
* 返回值:
*   0: 成功
*   -1: 客戶不存在
*   -2: 訂單項目無效
* 
* 使用範例:
*   DECLARE @OrderID INT;
*   EXEC usp_Order_Insert_CustomerOrder 
*       @CustomerID = 1001,
*       @OrderDate = '2026-01-01',
*       @OrderItems = '<Items>...</Items>',
*       @OrderIDOutput = @OrderID OUTPUT;
* 
* 修改歷程:
*   2026-01-15 張三 - 新增交易處理
*   2026-01-20 李四 - 優化索引使用
*******************************************************************************/
```

**原因：**
- 完整的註解讓其他開發人員快速理解 SP 的功能和使用方式
- 修改歷程有助於追蹤變更和問題排查
- 使用範例降低誤用的可能性

### 2. 程式碼段落註解

**規範：**
- 複雜邏輯前必須加上解釋性註解
- 使用單行註解 `--` 或多行註解 `/* */`
- 註解應說明「為什麼」而非「做什麼」

**正確範例：**
```sql
-- 使用 NOLOCK 提示以避免阻塞報表查詢，此查詢容許髒讀
SELECT * FROM Orders WITH (NOLOCK) WHERE Status = 'Pending';

/* 
 * 分批處理大量資料以避免交易記錄檔膨脹
 * 每批次處理 1000 筆記錄
 */
WHILE @RowCount > 0
BEGIN
    -- 處理邏輯
END
```

---

## 效能優化規範

### 1. 使用 SET NOCOUNT ON（強制）

**規範：**
每個 Stored Procedure 開頭必須加上 `SET NOCOUNT ON`

```sql
CREATE PROCEDURE usp_Example
AS
BEGIN
    SET NOCOUNT ON;  -- 強制要求
    
    -- 程式碼邏輯
END
```

**原因：**
- 抑制「受影響的列數」訊息，減少網路流量
- 顯著提升效能，特別是在多次查詢的 SP 中
- 避免觸發某些應用程式框架的不必要事件

### 2. 避免使用 SELECT *（強制禁止）

**規範：**
❌ 禁止使用 `SELECT *`，必須明確指定欄位

**錯誤範例：**
```sql
SELECT * FROM Customers WHERE CustomerID = @CustomerID;  -- ❌ 禁止
```

**正確範例：**
```sql
SELECT CustomerID, CustomerName, Email, Phone 
FROM Customers 
WHERE CustomerID = @CustomerID;  -- ✅ 正確
```

**原因：**
- `SELECT *` 會返回所有欄位，包含不需要的資料，浪費記憶體和網路頻寬
- 當資料表結構變更時，可能導致應用程式錯誤
- 無法有效利用涵蓋索引（Covering Index）
- 增加 IO 和 CPU 負擔

### 3. 使用合適的 JOIN 條件

**規範：**
- 必須在 ON 條件中指定 JOIN 條件，不可在 WHERE 中混用
- 優先使用 INNER JOIN 而非舊式 JOIN 語法
- JOIN 欄位必須有適當的索引

**錯誤範例：**
```sql
-- ❌ 舊式 JOIN 語法（禁止）
SELECT o.OrderID, c.CustomerName
FROM Orders o, Customers c
WHERE o.CustomerID = c.CustomerID;
```

**正確範例：**
```sql
-- ✅ 使用標準 JOIN 語法
SELECT o.OrderID, c.CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate >= '2026-01-01';
```

**原因：**
- 舊式語法可讀性差，容易導致意外的笛卡爾積
- 標準 JOIN 語法讓查詢優化器更容易產生最佳執行計畫
- 分離 JOIN 條件和篩選條件提高可維護性

### 4. 謹慎使用 Cursor（盡量避免）

**規範：**
❌ 原則上禁止使用 Cursor，優先使用 SET-based 操作

**錯誤範例：**
```sql
-- ❌ 使用 Cursor（效能極差）
DECLARE @CustomerID INT;
DECLARE cursor_customer CURSOR FOR
    SELECT CustomerID FROM Customers;
    
OPEN cursor_customer;
FETCH NEXT FROM cursor_customer INTO @CustomerID;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- 逐行處理
    UPDATE Orders SET Status = 'Processed' WHERE CustomerID = @CustomerID;
    FETCH NEXT FROM cursor_customer INTO @CustomerID;
END

CLOSE cursor_customer;
DEALLOCATE cursor_customer;
```

**正確範例：**
```sql
-- ✅ 使用 SET-based 操作
UPDATE o
SET o.Status = 'Processed'
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID;
```

**原因：**
- Cursor 是逐行處理，效能比 SET-based 操作慢 10-100 倍
- 佔用更多資源（記憶體、鎖定）
- 違反 SQL 的集合運算設計理念
- **例外情況：** 只有在邏輯上必須逐行處理且無法用 SET-based 實現時，才允許使用（需註明原因）

### 5. 參數化查詢（強制）

**規範：**
❌ 禁止使用動態 SQL 串接參數（SQL Injection 風險）

**錯誤範例：**
```sql
-- ❌ 高度危險！SQL Injection 風險
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = 'SELECT * FROM Users WHERE Username = ''' + @Username + '''';
EXEC(@SQL);
```

**正確範例：**
```sql
-- ✅ 使用參數化查詢
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = 'SELECT UserID, Username, Email FROM Users WHERE Username = @Username';
EXEC sp_executesql @SQL, N'@Username NVARCHAR(50)', @Username;
```

**原因：**
- 防止 SQL Injection 攻擊
- 允許執行計畫重用，提升效能
- 參數類型驗證

### 6. 適當使用 Transaction

**規範：**
- 必須使用 TRY-CATCH 包裹交易
- 交易範圍盡可能小
- 使用適當的隔離層級

**正確範例：**
```sql
CREATE PROCEDURE usp_Order_Insert_WithDetails
    @CustomerID INT,
    @OrderItems XML
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 插入訂單主檔
        INSERT INTO Orders (CustomerID, OrderDate)
        VALUES (@CustomerID, GETDATE());
        
        DECLARE @OrderID INT = SCOPE_IDENTITY();
        
        -- 插入訂單明細
        INSERT INTO OrderDetails (OrderID, ProductID, Quantity)
        SELECT @OrderID, Item.value('ProductID[1]', 'INT'), Item.value('Quantity[1]', 'INT')
        FROM @OrderItems.nodes('/Items/Item') AS T(Item);
        
        COMMIT TRANSACTION;
        RETURN 0;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
            
        -- 記錄錯誤
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
        RETURN -1;
    END CATCH
END
```

**原因：**
- 確保資料一致性
- 避免長時間鎖定資源
- 自動回滾錯誤操作

### 7. 避免在 WHERE 條件中使用函數

**規範：**
❌ 禁止在索引欄位上使用函數（導致索引失效）

**錯誤範例：**
```sql
-- ❌ 索引失效
SELECT * FROM Orders 
WHERE YEAR(OrderDate) = 2026;

-- ❌ 索引失效
SELECT * FROM Customers 
WHERE UPPER(Email) = 'TEST@EXAMPLE.COM';
```

**正確範例：**
```sql
-- ✅ 可使用索引
SELECT OrderID, CustomerID, OrderDate, TotalAmount 
FROM Orders 
WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01';

-- ✅ 使用計算欄位或調整查詢
SELECT CustomerID, CustomerName, Email 
FROM Customers 
WHERE Email = 'test@example.com';  -- 假設資料已標準化
```

**原因：**
- 在欄位上使用函數會導致索引掃描（Index Scan）而非索引搜尋（Index Seek）
- 大幅降低查詢效能，特別是大型資料表
- 增加 CPU 使用率

### 8. 使用 EXISTS 而非 IN（大型子查詢）

**規範：**
當子查詢返回大量資料時，使用 `EXISTS` 而非 `IN`

**較差範例：**
```sql
-- ⚠️ IN 在大量資料時效能較差
SELECT CustomerID, CustomerName
FROM Customers
WHERE CustomerID IN (SELECT CustomerID FROM Orders WHERE OrderDate >= '2026-01-01');
```

**最佳範例：**
```sql
-- ✅ EXISTS 效能更好（短路邏輯）
SELECT c.CustomerID, c.CustomerName
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o 
    WHERE o.CustomerID = c.CustomerID 
    AND o.OrderDate >= '2026-01-01'
);
```

**原因：**
- `EXISTS` 找到第一個匹配即停止（短路邏輯）
- `IN` 需要處理完整的子查詢結果
- 在大型資料集上效能差異顯著

### 9. 合理使用索引提示（謹慎使用）

**規範：**
⚠️ 僅在經過測試驗證後才使用索引提示

**範例：**
```sql
-- ⚠️ 需經過執行計畫驗證
SELECT CustomerID, CustomerName
FROM Customers WITH (INDEX(IX_Customer_Email))
WHERE Email = @Email;
```

**原因：**
- 過度使用索引提示可能限制查詢優化器
- 當資料分佈變化時，硬編碼的提示可能導致效能惡化
- 僅在確定查詢優化器選擇錯誤計畫時使用

### 10. NOLOCK 使用指引（謹慎使用）

**規範：**
⚠️ `WITH (NOLOCK)` 僅用於特定情境，必須註明使用原因

**允許使用的情境：**
- ✅ 報表查詢（容許髒讀的統計資料）
- ✅ 儀表板即時數據顯示
- ✅ 非交易性的資料瀏覽

**禁止使用的情境：**
- ❌ 金額計算（可能讀到未提交的交易）
- ❌ 庫存扣減（可能導致超賣）
- ❌ 任何 INSERT、UPDATE、DELETE 操作
- ❌ 需要精確一致性的查詢

**正確範例：**
```sql
-- ✅ 報表查詢可使用 NOLOCK
-- 註明：此查詢用於統計報表，容許髒讀以避免阻塞 OLTP 操作
SELECT 
    COUNT(*) AS TodayOrderCount,
    SUM(TotalAmount) AS TodayRevenue
FROM dbo.Orders WITH (NOLOCK)
WHERE OrderDate >= CAST(GETDATE() AS DATE);

-- ✅ 查詢操作使用 NOLOCK
SELECT CustomerID, CustomerName, Email
FROM dbo.Customers WITH (NOLOCK)
WHERE CustomerID = @CustomerID;
```

**錯誤範例：**
```sql
-- ❌ 編輯/刪除操作不使用 NOLOCK
UPDATE dbo.Orders WITH (NOLOCK)  -- ❌ 根本無效，UPDATE 會自動鎖定
SET Status = 'Completed'
WHERE OrderID = @OrderID;

-- ❌ 金額計算不可使用 NOLOCK
SELECT SUM(Amount) AS AccountBalance
FROM dbo.Transactions WITH (NOLOCK)  -- ❌ 可能讀到未提交的交易
WHERE CustomerID = @CustomerID;
```

**原因：**
- `NOLOCK` 會導致髒讀（讀取未提交的資料）
- 可能讀到重複資料或遺失資料
- INSERT/UPDATE/DELETE 本身就會鎖定，使用 NOLOCK 無效
- 只有在明確知道風險且可接受時才使用

### 11. 效能目標（強制）

**規範：**
所有 Stored Procedure 必須符合以下效能標準：

| 查詢類型 | 執行時間目標 | 超過目標時的處理 |
|---------|-------------|----------------|
| OLTP 查詢（新增/更新/刪除） | < 1 秒 | 優化索引、減少 JOIN |
| 簡單查詢（單表查詢） | < 2 秒 | 檢查執行計畫 |
| 複雜查詢（多表 JOIN） | < 5 秒 | 考慮增加索引、使用暫存表 |
| 報表查詢 | < 10 秒 | 考慮資料分割、快取 |
| 批次處理 | < 5 分鐘 | 拆分批次、使用分批處理 |

**檢查方式：**
```sql
-- 開啟執行時間統計
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

EXEC usp_YourProcedure @Param1 = 'Value';

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

**超過目標時的處理步驟：**
1. 查看實際執行計畫（SSMS: Ctrl+M）
2. 檢查是否使用索引（Index Seek vs Index Scan）
3. 檢查是否有隱式轉換
4. 考慮增加適當的索引
5. 考慮使用臨時表分段處理
6. 與 DBA 討論資料分割策略

### 12. 大量資料處理（臨時表 vs 資料表變數）

**規範：**
根據資料量選擇適當的暫存方式

**資料表變數（適用於小量資料）：**
```sql
-- ✅ 資料量 < 1000 筆時使用
DECLARE @TempOrders TABLE (
    OrderID INT,
    CustomerID INT,
    OrderDate DATETIME
);

INSERT INTO @TempOrders (OrderID, CustomerID, OrderDate)
SELECT TOP 500 OrderID, CustomerID, OrderDate
FROM dbo.Orders
WHERE Status = 'Pending';
```

**臨時資料表（適用於大量資料）：**
```sql
-- ✅ 資料量 > 1000 筆時使用臨時表
CREATE TABLE #TempOrders (
    OrderID INT,
    CustomerID INT,
    OrderDate DATETIME,
    PRIMARY KEY (OrderID)  -- 建立主鍵
);

INSERT INTO #TempOrders (OrderID, CustomerID, OrderDate)
SELECT OrderID, CustomerID, OrderDate
FROM dbo.Orders
WHERE OrderDate >= '2026-01-01';

-- 建立索引以提升後續查詢效能
CREATE INDEX IX_TempOrders_CustomerID ON #TempOrders(CustomerID);

-- 使用臨時表
SELECT t.OrderID, c.CustomerName
FROM #TempOrders t
INNER JOIN dbo.Customers c ON t.CustomerID = c.CustomerID;

-- 清理（SP 結束時自動清理）
DROP TABLE #TempOrders;
```

**Log 表處理的特殊考量：**
```sql
-- ✅ Log 表通常資料量大，先過濾到臨時表再處理
CREATE TABLE #RecentLogs (
    LogID INT,
    LogDate DATETIME,
    Message NVARCHAR(MAX)
);

-- 先過濾最近的資料到臨時表
INSERT INTO #RecentLogs
SELECT LogID, LogDate, Message
FROM dbo.SystemLog WITH (NOLOCK)  -- Log 表可使用 NOLOCK
WHERE LogDate >= DATEADD(DAY, -7, GETDATE());

-- 再對臨時表進行複雜查詢
SELECT LogDate, COUNT(*) AS ErrorCount
FROM #RecentLogs
WHERE Message LIKE '%Error%'
GROUP BY LogDate;

DROP TABLE #RecentLogs;
```

**原因：**
- 資料表變數無統計資訊，大量資料時執行計畫可能不佳
- 臨時表有統計資訊，優化器能產生更好的執行計畫
- 臨時表可以建立索引，大幅提升查詢效能
- Log 表通常資料量極大，使用臨時表過濾可避免鎖定

---

## 資料操作規範

### 1. 刪除後新增的使用情境

**規範：**
⚠️ 「先刪除後新增」僅適用於特定情境，需註明原因

**允許使用的情境：**
- ✅ 臨時資料表的重建
- ✅ 無外鍵約束的設定表
- ✅ 批次資料完全替換（如每日匯入）

**禁止使用的情境：**
- ❌ 有外鍵約束的主表
- ❌ 有 IDENTITY 欄位的表（浪費 ID）
- ❌ 有觸發器的表（觸發兩次）
- ❌ 部分資料更新

**錯誤範例：**
```sql
-- ❌ 不建議：有外鍵約束且浪費 IDENTITY
DELETE FROM dbo.OrderDetails WHERE OrderID = @OrderID;
INSERT INTO dbo.OrderDetails (OrderID, ProductID, Quantity)
VALUES (@OrderID, @ProductID, @Quantity);
```

**正確範例：**
```sql
-- ✅ 使用 MERGE 語句
MERGE INTO dbo.OrderDetails AS Target
USING (
    SELECT @OrderID AS OrderID, @ProductID AS ProductID, @Quantity AS Quantity
) AS Source
ON Target.OrderID = Source.OrderID AND Target.ProductID = Source.ProductID
WHEN MATCHED THEN 
    UPDATE SET Quantity = Source.Quantity
WHEN NOT MATCHED THEN 
    INSERT (OrderID, ProductID, Quantity) 
    VALUES (Source.OrderID, Source.ProductID, Source.Quantity);

-- ✅ 或使用條件式更新/插入
IF EXISTS (SELECT 1 FROM dbo.OrderDetails WHERE OrderID = @OrderID AND ProductID = @ProductID)
BEGIN
    UPDATE dbo.OrderDetails 
    SET Quantity = @Quantity 
    WHERE OrderID = @OrderID AND ProductID = @ProductID;
END
ELSE
BEGIN
    INSERT INTO dbo.OrderDetails (OrderID, ProductID, Quantity)
    VALUES (@OrderID, @ProductID, @Quantity);
END
```

**臨時表的例外（允許）：**
```sql
-- ✅ 臨時表可以先刪後增
IF OBJECT_ID('tempdb..#TempData') IS NOT NULL
    DROP TABLE #TempData;

CREATE TABLE #TempData (
    ID INT,
    Value NVARCHAR(100)
);
```

**原因：**
- 避免觸發外鍵約束錯誤
- 避免浪費 IDENTITY 序號
- 避免觸發器被執行兩次
- MERGE 或條件式操作更高效且邏輯更清晰

### 2. Log 記錄規範（強制）

**規範：**
- 所有包含資料異動的 SP 必須記錄操作日誌
- 錯誤必須記錄到 ErrorLog 表
- 重要操作必須記錄到 AuditLog 表

**Log 記錄檢查事項：**
1. ✅ 是否正確寫入 Log 表？
2. ✅ Log 訊息是否包含足夠的除錯資訊？
3. ✅ 是否記錄操作人員、時間、操作內容？
4. ✅ 錯誤 Log 是否包含錯誤詳情？

**正確範例：**
```sql
CREATE PROCEDURE usp_Order_Update_Status
    @OrderID INT,
    @NewStatus NVARCHAR(20),
    @OperatorID INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @OldStatus NVARCHAR(20);
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 取得舊狀態
        SELECT @OldStatus = Status 
        FROM dbo.Orders 
        WHERE OrderID = @OrderID;
        
        -- 更新狀態
        UPDATE dbo.Orders
        SET Status = @NewStatus, UpdatedDate = GETDATE()
        WHERE OrderID = @OrderID;
        
        -- ✅ 記錄操作日誌
        INSERT INTO dbo.AuditLog (TableName, RecordID, Operation, OldValue, NewValue, OperatorID, OperatedDate)
        VALUES (
            'Orders', 
            @OrderID, 
            'UPDATE_STATUS', 
            @OldStatus, 
            @NewStatus, 
            @OperatorID, 
            GETDATE()
        );
        
        COMMIT TRANSACTION;
        RETURN 0;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- ✅ 記錄錯誤日誌
        INSERT INTO dbo.ErrorLog (ProcedureName, ErrorMessage, ErrorLine, ErrorSeverity, OperatedDate)
        VALUES (
            'usp_Order_Update_Status',
            ERROR_MESSAGE(),
            ERROR_LINE(),
            ERROR_SEVERITY(),
            GETDATE()
        );
        
        RAISERROR('訂單狀態更新失敗：%s', 16, 1, ERROR_MESSAGE());
        RETURN -1;
    END CATCH
END
```

**原因：**
- 提供完整的操作追蹤（Audit Trail）
- 便於問題排查和除錯
- 符合資安和合規要求
- 支援資料復原

### 3. 變更影響分析（強制）

**規範：**
修改現有 SP 前必須進行影響分析

**影響分析檢查清單：**
- [ ] 是否變更現有參數（名稱、型別、順序）？
- [ ] 是否變更返回值或 OUTPUT 參數？
- [ ] 是否變更返回的資料集結構（欄位增減）？
- [ ] 是否有其他 SP 呼叫此 SP？
- [ ] 是否有應用程式直接呼叫此 SP？
- [ ] 是否有排程作業使用此 SP？
- [ ] 是否改變交易邏輯或鎖定行為？

**查找相依性的 SQL：**
```sql
-- 查找哪些 SP 呼叫此 SP
SELECT DISTINCT
    OBJECT_NAME(referencing_id) AS ReferencingSP,
    OBJECT_NAME(referenced_id) AS ReferencedSP
FROM sys.sql_expression_dependencies
WHERE referenced_id = OBJECT_ID('dbo.usp_YourProcedure')
    AND referencing_id IS NOT NULL;
```

**範例：**
```sql
/*
 * 變更影響分析：
 * - 新增參數 @IncludeDetails BIT（預設值 0，向後相容）
 * - 不影響現有呼叫者
 * - 相依 SP：usp_Report_GetOrders（已測試）
 */
ALTER PROCEDURE usp_Order_Get_ByCustomer
    @CustomerID INT,
    @IncludeDetails BIT = 0  -- 新參數使用預設值
AS
BEGIN
    -- 實作邏輯
END
```

**原因：**
- 避免破壞現有功能
- 減少生產環境錯誤
- 確保完整測試覆蓋

---

## 禁止使用的做法

### 1. ❌ 禁止在 WHERE 條件中使用 OR 連接不同欄位

**錯誤範例：**
```sql
-- ❌ 導致索引無法有效使用
SELECT * FROM Orders 
WHERE CustomerID = @CustomerID OR OrderDate = @OrderDate;
```

**正確範例：**
```sql
-- ✅ 拆分為 UNION 查詢
SELECT OrderID, CustomerID, OrderDate FROM Orders WHERE CustomerID = @CustomerID
UNION
SELECT OrderID, CustomerID, OrderDate FROM Orders WHERE OrderDate = @OrderDate;
```

**原因：**
- OR 條件可能導致索引掃描
- UNION 允許每個查詢使用最佳索引

### 2. ❌ 禁止在大型資料表上使用 NOT IN

**錯誤範例：**
```sql
-- ❌ 效能極差且 NULL 值處理問題
SELECT * FROM Customers 
WHERE CustomerID NOT IN (SELECT CustomerID FROM Orders);
```

**正確範例：**
```sql
-- ✅ 使用 LEFT JOIN 和 NULL 檢查
SELECT c.CustomerID, c.CustomerName
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE o.CustomerID IS NULL;

-- ✅ 或使用 NOT EXISTS
SELECT c.CustomerID, c.CustomerName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerID = c.CustomerID
);
```

**原因：**
- `NOT IN` 在遇到 NULL 值時可能產生非預期結果
- 大型資料集上效能極差
- `NOT EXISTS` 或 `LEFT JOIN` 效能更好且邏輯更清晰

### 3. ❌ 禁止使用 WHILE LOOP 處理大量資料

**錯誤範例：**
```sql
-- ❌ 逐筆更新，效能極差
DECLARE @Counter INT = 1;
WHILE @Counter <= 10000
BEGIN
    UPDATE Products SET Price = Price * 1.1 WHERE ProductID = @Counter;
    SET @Counter = @Counter + 1;
END
```

**正確範例：**
```sql
-- ✅ 單一 SET-based 更新
UPDATE Products 
SET Price = Price * 1.1 
WHERE ProductID BETWEEN 1 AND 10000;
```

**原因：**
- WHILE LOOP 逐行處理，效能極差
- SET-based 操作利用批次處理，效能提升數十倍

### 4. ❌ 禁止在 SP 中使用 SELECT 返回訊息（除非是資料查詢）

**錯誤範例：**
```sql
-- ❌ 混淆返回值
CREATE PROCEDURE usp_Order_Insert
AS
BEGIN
    INSERT INTO Orders (...) VALUES (...);
    SELECT 'Insert Success';  -- ❌ 不應這樣做
END
```

**正確範例：**
```sql
-- ✅ 使用 RETURN 或 OUTPUT 參數
CREATE PROCEDURE usp_Order_Insert
    @OrderIDOutput INT OUTPUT
AS
BEGIN
    INSERT INTO Orders (...) VALUES (...);
    SET @OrderIDOutput = SCOPE_IDENTITY();
    RETURN 0;  -- 0 表示成功
END
```

**原因：**
- SELECT 訊息難以在應用程式中處理
- 返回值或 OUTPUT 參數更清晰且易於程式處理
- 避免與資料查詢混淆

---

## 錯誤處理規範

### 1. 必須使用 TRY-CATCH（強制）

**規範：**
所有可能發生錯誤的操作必須包裹在 TRY-CATCH 中

**正確範例：**
```sql
CREATE PROCEDURE usp_Order_Update
    @OrderID INT,
    @Status NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        -- 驗證訂單存在
        IF NOT EXISTS (SELECT 1 FROM Orders WHERE OrderID = @OrderID)
        BEGIN
            RAISERROR('訂單不存在 (OrderID: %d)', 16, 1, @OrderID);
            RETURN -1;
        END
        
        -- 更新訂單
        UPDATE Orders 
        SET Status = @Status, UpdatedDate = GETDATE()
        WHERE OrderID = @OrderID;
        
        RETURN 0;
    END TRY
    BEGIN CATCH
        -- 記錄錯誤資訊
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        -- 可選：記錄到錯誤日誌表
        INSERT INTO ErrorLog (ProcedureName, ErrorMessage, ErrorDate)
        VALUES ('usp_Order_Update', @ErrorMessage, GETDATE());
        
        -- 重新拋出錯誤
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
        RETURN -99;
    END CATCH
END
```

**原因：**
- 防止未處理的錯誤導致應用程式崩潰
- 提供錯誤追蹤和除錯資訊
- 確保交易正確回滾

### 2. 錯誤訊息必須有意義

**規範：**
錯誤訊息應清楚說明問題和相關參數

**錯誤範例：**
```sql
RAISERROR('Error', 16, 1);  -- ❌ 訊息無意義
```

**正確範例：**
```sql
RAISERROR('無法更新訂單狀態。訂單ID: %d, 原因: 訂單已完成', 16, 1, @OrderID);
```

---

## 安全性規範

### 1. 使用參數化查詢（強制）

已在效能規範中說明，同時也是最重要的安全規範。

### 2. 限制權限

**規範：**
- Stored Procedure 應使用 `EXECUTE AS` 指定執行身份
- 遵循最小權限原則

**範例：**
```sql
CREATE PROCEDURE usp_Report_GetSales
WITH EXECUTE AS 'ReportUser'  -- 使用特定權限執行
AS
BEGIN
    -- 查詢邏輯
END
```

### 3. 驗證輸入參數（強制）

**規範：**
所有輸入參數必須經過驗證

**正確範例：**
```sql
CREATE PROCEDURE usp_Order_GetByID
    @OrderID INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 驗證參數
    IF @OrderID IS NULL OR @OrderID <= 0
    BEGIN
        RAISERROR('參數 @OrderID 必須為正整數', 16, 1);
        RETURN -1;
    END
    
    -- 查詢邏輯
    SELECT OrderID, CustomerID, OrderDate, TotalAmount
    FROM Orders
    WHERE OrderID = @OrderID;
END
```

**原因：**
- 防止無效資料導致錯誤
- 提早發現問題
- 提升應用程式穩定性

---

## 程式碼品質規範

### 1. 程式碼格式化

**規範：**
- 使用一致的縮排（建議 4 個空格）
- 關鍵字使用大寫
- 適當的空行分隔邏輯區塊

**正確範例：**
```sql
CREATE PROCEDURE usp_Order_Insert
    @CustomerID INT,
    @OrderDate DATETIME,
    @OrderIDOutput INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 插入訂單
        INSERT INTO Orders (CustomerID, OrderDate, Status)
        VALUES (@CustomerID, @OrderDate, 'Pending');
        
        -- 取得新訂單ID
        SET @OrderIDOutput = SCOPE_IDENTITY();
        
        COMMIT TRANSACTION;
        RETURN 0;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
            
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
        RETURN -1;
    END CATCH
END
```

### 2. 避免過長的 Stored Procedure

**規範：**
- 單一 SP 不應超過 300 行
- 複雜邏輯應拆分為多個 SP
- 遵循單一職責原則

**原因：**
- 提高可讀性和可維護性
- 便於測試和除錯
- 促進程式碼重用

### 3. 版本控制

**規範：**
- 所有 SP 必須納入版本控制系統（Git）
- 使用 ALTER 而非 DROP/CREATE（保留權限）
- 在標頭註解中記錄修改歷程

---

## 檢查清單

在提交 Stored Procedure 前，請確認：

- [ ] 已加上 `usp_` 前綴
- [ ] 包含完整的標頭註解（作者、說明、參數、返回值、使用範例）
- [ ] 使用 `SET NOCOUNT ON`
- [ ] 沒有使用 `SELECT *`
- [ ] 沒有使用舊式 JOIN 語法
- [ ] 沒有在索引欄位上使用函數
- [ ] 使用 TRY-CATCH 錯誤處理
- [ ] 所有動態 SQL 都使用參數化查詢
- [ ] 沒有使用 Cursor（或已註明必要原因）
- [ ] Transaction 有適當的錯誤處理和 ROLLBACK
- [ ] 輸入參數已驗證
- [ ] 程式碼已格式化且可讀性良好
- [ ] 已測試執行計畫，確認使用適當的索引
- [ ] 錯誤訊息清楚且有意義

---

## 參考資源

- **執行計畫分析：** 使用 SQL Server Management Studio 的「顯示實際執行計畫」功能
- **效能監控：** 使用 `SET STATISTICS IO ON` 和 `SET STATISTICS TIME ON`
- **最佳實踐：** Microsoft SQL Server 官方文檔

---

> [!CAUTION]
> 違反本規範可能導致：
> - 嚴重的效能問題（查詢執行時間過長、系統資源耗盡）
> - SQL Injection 安全漏洞
> - 資料庫死鎖
> - 資料不一致
> - 系統穩定性問題
> 
> 請務必嚴格遵守本規範。如有疑問，請諮詢 DBA 或資深開發人員。
