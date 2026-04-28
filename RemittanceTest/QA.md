# 學習筆記 Q&A

## 依賴注入（DI）

**Q: DI 容器是什麼？**
DI 容器是框架內部專門管理「誰需要什麼、給誰什麼」的地方。你只需要「註冊」和「使用」，框架負責建立物件和管理生命週期。

**Q: 把 Service 註冊到 DI 容器是什麼意思？**
就是告訴框架：「當有人需要 `IRemittanceService`，請幫我 `new RemittanceService()` 給他。」
沒有註冊的話，框架找不到對應的實作，Controller 要求注入時會直接報錯。

**Q: Controller 也需要註冊嗎？**
不需要，`builder.Services.AddControllers()` 已經幫你自動掃描所有 Controller。
只有 Service 需要手動註冊。

**Q: Service 有幾種生命週期？差異是什麼？**

| 方法 | 同一個 Request 內 | 不同 Request 之間 | 說明 |
|------|---|---|------|
| `AddScoped` | 共用同一個實例 | 各自獨立的新實例 | 最常用，一般 Service 都用這個 |
| `AddTransient` | 每次注入都是新實例 | 每次注入都是新實例 | 適合輕量、無狀態的工具類別 |
| `AddSingleton` | 共用同一個實例 | 共用同一個實例 | 適合共用設定、Cache 等全域資源 |

餐廳比喻：
- **AddScoped**：每個客人（Request）有自己的一份，用完就丟，下一個客人再來一杯新的
- **AddTransient**：每次跟服務生要，就給一份新的，即使同一次用餐要了很多次
- **AddSingleton**：整間餐廳共用一台咖啡機，開到關都是同一台

**Q: 介面（Interface）和類別（Class）的差異？**

- **介面**：只定義「要有什麼」，不寫實作（合約）
- **類別**：實際寫出「怎麼做」（履行合約的人）

```csharp
// 介面 — 只定義規格
interface IRemittanceService {
    (bool IsSuccess, string Message) CancelRemittance(int id);
}

// 類別 — 實際實作
class RemittanceService : IRemittanceService {
    public (bool IsSuccess, string Message) CancelRemittance(int id) {
        // 實際邏輯寫在這裡
    }
}
```

好處：Controller 只依賴介面，未來換實作（例如從 In-Memory 換成 SQL Server）只需要改 DI 註冊那一行，Controller 完全不用動。

```csharp
// 只改這一行，Controller 不需要動
builder.Services.AddScoped<IRemittanceService, RemittanceServiceDB>();
```

**Q: `(bool IsSuccess, string Message) CancelRemittance(int id)` 這寫法是什麼意思？**

這是 C# 的 **Tuple 回傳值**，方法同時回傳兩個值：

```csharp
// C# — 回傳型別放方法名稱前面
(bool IsSuccess, string Message)  CancelRemittance  (int id)
        ↑ 回傳型別                    ↑ 方法名稱        ↑ 參數
```

對應 TypeScript 的寫法：
```typescript
// TypeScript — 回傳型別放後面
cancelRemittance(id: number): { isSuccess: boolean, message: string }
```

使用時：
```csharp
var result = CancelRemittance(1);
result.IsSuccess // true or false
result.Message   // "取消成功" 或 "狀態不符"
```

回傳方式：
```csharp
return (true, "取消成功");
return (false, "狀態不符，無法取消");
return (false, "找不到此筆資料");
```

**Q: `private static readonly List<Remittance> _db = new()` 這段是什麼意思？**

| 關鍵字 | 意思 |
|--------|------|
| `private` | 只有這個 class 內部可以存取 |
| `static` | 所有實例共用同一份資料 |
| `readonly` | 這個變數不能被重新指派（但內容可以改） |
| `List<Remittance>` | 存放 `Remittance` 物件的清單 |
| `new()` | 建立新的 List，型別從左邊自動推斷 |

**Q: `new()` 後面的 `{ }` 是什麼？**

這是**集合初始化語法**，建立 List 的同時直接塞入初始資料：

```csharp
// C#
private static readonly List<Remittance> _db = new()
{
    new Remittance { Id = 1, AccountName = "測試企業A", Status = 0 },
    new Remittance { Id = 2, AccountName = "測試企業B", Status = 1 },
};
```

對應 TypeScript：
```typescript
const db: Remittance[] = [
  { id: 1, accountName: "測試企業A", status: 0 },
  { id: 2, accountName: "測試企業B", status: 1 },
];
```

C# 每筆資料都要 `new Remittance { }`，因為是強型別，必須明確告訴編譯器這是哪個型別的實例。TypeScript 直接寫物件字面值就好。

**Q: `static` 是共用的意思嗎？**

對，`static` 屬於 **class 本身，不屬於任何實例**。不管 `new` 幾個實例出來，`static` 的東西永遠只有一份，大家共用同一個。

```csharp
// 沒有 static — 每個實例有自己的 _db
RemittanceService instance1 = new(); // 有自己的 _db
RemittanceService instance2 = new(); // 有自己的 _db，跟 instance1 不同

// 有 static — 所有實例共用同一個 _db
RemittanceService instance1 = new(); // 共用同一個 _db
RemittanceService instance2 = new(); // 同上，是同一份資料
```

這個專案用 `static` 是因為 `AddScoped` 每個 Request 會建立新的實例，若沒有 `static`，資料每次都會重置，無法模擬資料庫的持久化。

**Q: C# 的 `var` 和 JS 的 `var` 一樣嗎？**

概念類似但有差異：
- **JS** 的 `var`：弱型別，可以隨時換型別
- **C#** 的 `var`：型別推斷，編譯器自動判斷型別，但一旦確定就不能換

```csharp
var item = _db.FirstOrDefault(...); // 推斷為 Remittance
item = "hello"; // ❌ 不行，型別已確定
```

C# 的 `var` 只是少打字，型別還是強型別。

**Q: C# 字串用 `""` 還是 `''`？**

| 符號 | 型別 | 說明 |
|------|------|------|
| `""` | `string` | 字串 |
| `''` | `char` | 單一字元 |

```csharp
string name = "Josh";  // 字串
char letter = 'A';     // 單一字元
```

混用會編譯錯誤。`char` 通常用在處理字串內單一字元的情境，一般開發大多用 `string`。

**Q: 數字、字串、字元的差異？**

```csharp
int n = 9;      // 數字
string s = "9"; // 字串
char c = '9';   // 字元（char）
```

三個是完全不同的型別。需要轉換時：
```csharp
int.Parse("9")  // 字串轉數字
9.ToString()    // 數字轉字串
```

**Q: C# 是多執行緒嗎？`lock` 是什麼？**

對，C# 天生支援多執行緒。ASP.NET Core 使用**執行緒池（Thread Pool）**，預先建好一批執行緒重複使用，多個同時進來的 Request 會各自佔用一條執行緒**真的同時執行**。

```
Request A → 執行緒 1 ─┐
Request B → 執行緒 2 ─┼─ 同時存取 _db  ← 併發問題
Request C → 執行緒 3 ─┘
```

`lock` 是 C# 內建關鍵字，確保同一時間只有一條執行緒可以進入這段程式碼：

```csharp
lock (_lockObj)
{
    // 同一時間只有一條執行緒在這裡執行
    // 其他執行緒在外面排隊等待
}
```

排隊流程：
1. 執行緒 1 進入 `lock` → 鎖上
2. 執行緒 2、3 想進來 → 被擋在外面排隊
3. 執行緒 1 執行完離開 → 解鎖
4. 執行緒 2 進入 → 鎖上，依此類推

**Q: JS 單執行緒為什麼也會有併發問題？**

JS 的「併發」和 C# 不同：
- **JS**：單執行緒 + Event Loop，併發問題發生在 `await` 讓出控制權的空隙
- **C#**：多條執行緒真的同時執行

```javascript
// JS — await 讓出控制權的空隙造成問題
async function cancel(id) {
    const item = await db.findById(id); // 等待期間讓出控制權
    if (item.status === 0) {
        item.status = 9; // 兩個請求都可能跑到這裡
    }
}
```

本質不同，但結果一樣會有競爭問題。

**Q: `[ApiController]` 是陣列嗎？**

不是，這是 C# 的 **Attribute（屬性標記）**，用 `[]` 包起來放在 class 或方法上面，告訴框架這個 class 有什麼特性或行為。類比 TypeScript 的 Decorator（`@`）：

```typescript
// TypeScript Decorator
@Controller('api/remittance')
export class RemittanceController {}
```

```csharp
// C# Attribute
[ApiController]
[Route("api/[controller]")]
public class RemittanceController : ControllerBase {}
```

`[ApiController]` 有實際功能，會自動幫你：
- 驗證 Model 輸入，失敗直接回傳 400
- 自動從 Request Body 綁定參數

`[Route("api/[controller]")]` 定義路由，`[controller]` 會自動替換成 class 名稱去掉 Controller，所以路由是 `api/remittance`。

**Q: 建構子（Constructor）是什麼？**

建構子是**物件被建立時自動執行的方法**，用來初始化物件需要的東西。

```javascript
// JS
class RemittanceController {
    constructor(service) {
        this.service = service; // 建立時自動執行
    }
}
```

```csharp
// C# — 用 class 名稱當建構子，不用 constructor 關鍵字
public class RemittanceController : ControllerBase
{
    private readonly IRemittanceService _service;

    public RemittanceController(IRemittanceService service)
    {
        _service = service;
    }
}
```

**Q: Controller 怎麼拿到 RemittanceService 裡的邏輯？**

透過 DI 的整條流程：

```
Program.cs 註冊：AddScoped<IRemittanceService, RemittanceService>()
    ↓
Controller 建構子說「我要 IRemittanceService」
    ↓
DI 容器把 RemittanceService 塞進來
    ↓
Controller 透過 _service 呼叫你寫的邏輯
```

這是 ASP.NET Core 的標準規範，也是通用設計模式，NestJS、Spring、Angular 都一樣。

**Q: Controller 怎麼回傳 HTTP 狀態碼？**

ASP.NET Core 內建語法糖，直接用方法名稱回傳：

```csharp
return Ok(result.Message);       // 200
return BadRequest(result.Message); // 400
return NotFound(result.Message);   // 404
```

背後實際是：
```csharp
return new OkObjectResult(result.Message);
return new BadRequestObjectResult(result.Message);
return new NotFoundObjectResult(result.Message);
```

根據 Service 回傳結果判斷：
```csharp
var result = _service.CancelRemittance(id);
if (result.IsSuccess) return Ok(result.Message);
if (result.Message == "找不到此筆資料") return NotFound(result.Message);
return BadRequest(result.Message);
```

**Q: 真實資料庫環境下怎麼防 Race Condition？**

有兩種思路：

**樂觀鎖**（不鎖，寫入時檢查）— 適合衝突機率低的情境：

T-SQL：
```sql
UPDATE Remittances
SET Status = 9
WHERE Id = @id AND Status = 0
-- 影響 0 筆代表失敗（狀態已被改或找不到）
```

EF Core：
```csharp
var affected = await db.Remittances
    .Where(x => x.Id == id && x.Status == 0)
    .ExecuteUpdateAsync(x => x.SetProperty(r => r.Status, 9));

if (affected == 0) return (false, "狀態不符或找不到");
return (true, "取消成功");
```

**悲觀鎖**（查詢時就鎖住）— 適合衝突機率高的情境，但效能較差。

關鍵概念：把**條件放在更新裡**，讓資料庫保證查詢和更新是原子操作，不需要額外加鎖。

**Q: T-SQL 是什麼？**

T-SQL（Transact-SQL）是 Microsoft SQL Server 的 SQL 方言，在標準 SQL 上加了微軟自己擴充的語法。基本的 `SELECT`、`UPDATE`、`WHERE` 都一樣，部分語法是 SQL Server 特有的（例如 `TOP`、`GETDATE()`）。

**Q: EF Core 是什麼？**

Entity Framework Core，C# 的 ORM，類似 JS 的 Prisma / Sequelize，讓你用 C# 語法操作資料庫，背後自動產生 SQL。

---

## 專案架構

**Q: Program.cs 的作用？**
是程式進入點，類似 Express 的 `app.js`：
- `builder.Services.AddControllers()` → 把 Controller 註冊進 DI 容器
- `app.MapControllers()` → 把 Controller 的路由對應起來
- `builder.Services.AddScoped<>()` → 手動註冊 Service

**Q: 這個標準資料夾結構是 .NET 規範嗎？**
對，是 ASP.NET Core 標準結構：
| 資料夾 | 說明 | JS 類比 |
|--------|------|---------|
| `Controllers/` | 處理 HTTP 請求 | Express 的 `routes/` |
| `Services/` | 商業邏輯 | 自己寫的 `services/` |
| `Models/` | 資料結構 | TypeScript 的 `types/` |
| `Program.cs` | 進入點 + DI 註冊 | Express 的 `app.js` |

**Q: 哪些檔案是預設就有的？**
- `Program.cs`
- `appsettings.json`
- `appsettings.Development.json`
- `Properties/launchSettings.json`
- `Controllers/WeatherForecastController.cs`（範例，通常會刪掉）
- `WeatherForecast.cs`（範例，通常會刪掉）

**Q: wwwroot 是做什麼的？**
放前端靜態檔案的資料夾，瀏覽器可以直接存取。這個專案是前後端放在一起的架構（CSR），不是 SSR。大型專案通常前後端分開部署，`wwwroot` 就用不到了。

---

## .NET SDK 安裝

**Q: 要安裝哪個 SDK？**
安裝 `dotnet-sdk`，它包含 runtime + 工具鏈，裝一個就夠。
`dotnet-runtime` 只能執行，無法開發編譯。

```bash
# 安裝指定版本
brew install --cask dotnet-sdk@8

# 確認已安裝的版本
dotnet --list-sdks
```
