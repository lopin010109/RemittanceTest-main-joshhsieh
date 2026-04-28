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
