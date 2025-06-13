# GitBook：Android 串接 C# API 教學（無 Entity Framework）

***

## 1. ✅ 前置準備

| 項目          | 工具                              |
| ----------- | ------------------------------- |
| C# API 建立工具 | Visual Studio 2022+ (.NET 6 以上) |
| Android 工具  | Android Studio 4.0+             |
| 測試設備        | Android 模擬器 或 實體裝置              |
| 區網 IP       | 主機需有區網 IP，如 192.168.0.X         |

***

## 2. ✅ 建立 C# Web API（無 EF）

### 🔧 建立專案

1. Visual Studio ➜ 建立新專案 ➜ ASP.NET Core Web API
2. 設定：
   * 不使用 HTTPS
   * 不使用 OpenAPI
   * 不使用 Entity Framework

***

### 📁 Receiving Model

`Models/Receiving.cs`:

```csharp
public class Receiving {
    public string LotId { get; set; }
    public int Qty { get; set; }
    public string PartId { get; set; }
    public string SuggestionLoc { get; set; }
    public string Locator { get; set; }
    public string IsEnd { get; set; }
}
```

***

### 📂 Receiving Controller

`Controllers/ReceivingController.cs`:

```csharp
[ApiController]
[Route("[controller]")]
public class ReceivingController : ControllerBase {
    private static List<Receiving> receivingList = new();

    [HttpGet]
    public ActionResult<IEnumerable<Receiving>> Get() => receivingList;

    [HttpPost]
    public IActionResult Post([FromBody] Receiving receiving) {
        receivingList.Add(receiving);
        return Ok();
    }
}
```

***

### 🛠 Program.cs 設定

```csharp
builder.WebHost.UseUrls("http://192.168.0.7:5265"); // 改為你的區網 IP
```

***

### 🔥 Windows 開 Port

```bash
netsh advfirewall firewall add rule name="WebAPI" dir=in action=allow protocol=TCP localport=5265
```

***

## 3. ✅ 調整 API 網路可連性

### `launchSettings.json` 修改

```json
"applicationUrl": "http://192.168.0.7:5265"
```

***

## 4. ✅ 建立 Android Retrofit 呼叫

### ApiClient.java

```java
public class ApiClient {
    private static final String BASE_URL = "http://192.168.0.7:5265/";
    private static Retrofit retrofit;

    public static Retrofit getClient() {
        if (retrofit == null) {
            retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        }
        return retrofit;
    }
}
```

***

### ApiService.java

```java
public interface ApiService {
    @POST("receiving")
    Call<Void> postReceiving(@Body Receiving receiving);

    @GET("receiving")
    Call<List<Receiving>> getReceivingList();
}
```

***

### Receiving.java

```java
public class Receiving {
    private String lotId;
    private int qty;
    private String partId;
    private String suggestionLoc;
    private String locator;
    private String isEnd;

    public Receiving(String lotId, int qty, String partId, String suggestionLoc, String locator, String isEnd) {
        this.lotId = lotId;
        this.qty = qty;
        this.partId = partId;
        this.suggestionLoc = suggestionLoc;
        this.locator = locator;
        this.isEnd = isEnd;
    }
}
```

***

### 呼叫 POST / GET

```java
Receiving data = new Receiving("LOT001", 100, "PART123", "A01", "L01", "N");
ApiService apiService = ApiClient.getClient().create(ApiService.class);

// POST
apiService.postReceiving(data).enqueue(new Callback<Void>() {
    public void onResponse(Call<Void> call, Response<Void> response) {
        Log.i("API", "成功：" + response.code());
    }

    public void onFailure(Call<Void> call, Throwable t) {
        Log.e("API", "錯誤：" + t.getMessage());
    }
});

// GET
apiService.getReceivingList().enqueue(new Callback<List<Receiving>>() {
    public void onResponse(Call<List<Receiving>> call, Response<List<Receiving>> response) {
        List<Receiving> list = response.body();
        // 處理資料
    }

    public void onFailure(Call<List<Receiving>> call, Throwable t) {
        Log.e("API", "錯誤：" + t.getMessage());
    }
});
```

***

## 5. ✅ Android 網路設定

### `AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

***

### `network_security_config.xml`

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

`AndroidManifest.xml` 設定：

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

***

## 6. ✅ 測試與除錯

| 項目           | 檢查                     |
| ------------ | ---------------------- |
| 主機 IP        | 是否正確 (192.168.x.x)     |
| Android 可否連線 | 手機瀏覽器是否能打開 API         |
| API 啟動       | Console 有無啟動訊息         |
| Port 開放      | `netstat`, `telnet` 測試 |

***

## 7. ✅ 常見錯誤與解法

| 錯誤訊息                                    | 解法                           |
| --------------------------------------- | ---------------------------- |
| `EPERM Operation not permitted`         | 模擬器或裝置未允許 HTTP               |
| `Cleartext communication not permitted` | 加入 network\_security\_config |
| `404 Not Found`                         | 路徑錯誤 / API 名稱錯誤              |
| `Failed to connect`                     | IP 錯誤 / port 未開 / API 關閉     |
