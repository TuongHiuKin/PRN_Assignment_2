# AIVES — AI-powered Viva Exam System

Hệ thống thi vấn đáp (viva) có AI hỗ trợ: sinh viên nghe câu hỏi do máy đọc, trả lời bằng giọng nói,
hệ thống chuyển giọng nói thành văn bản và chấm điểm theo rubric. Giảng viên là người quyết định
điểm cuối cùng.

**Assignment 2 — PRN222 · ASP.NET Core 8 · SQL Server · 3 lớp**

---

## Kiến trúc 3 lớp

![Kiến trúc AIVES ASM2](docs/architecture/aives-asm2.png)

Sơ đồ nguồn: [`docs/architecture/aives-asm2.drawio`](docs/architecture/aives-asm2.drawio) — mở bằng
[draw.io](https://app.diagrams.net) hoặc extension Draw.io Integration trong VS Code.

### Luồng phụ thuộc

```
StudentNameMVC  →  AIVES.BLL  →  AIVES.DAL  →  SQL Server
   (Web App)        (.dll)        (.dll)
```

Chiều ngược lại không được phép: `AIVES.DAL` không tham chiếu `AIVES.BLL` hay `StudentNameMVC`;
`AIVES.BLL` không dùng `DbContext`, không biết HTTP/Razor.

| Tầng | Project | Nhiệm vụ |
| :--- | :--- | :--- |
| Presentation | `StudentNameMVC` | Razor Pages, SignalR Hub, Controllers, DI khởi động |
| Business Logic | `AIVES.BLL` | Orchestrator phiên thi, dịch vụ AI, tính toán báo cáo |
| Data Access | `AIVES.DAL` | Entity, `AIVESDbContext`, Repository (Scoped, EF Core) |
| Database | SQL Server | `AIVES` — bảng viva, account, rubric |

### Quy ước tầng

- **Presentation** — chỉ nhận HTTP input, kiểm tra `ModelState`, phân quyền, map ViewModel. Không gọi
  `DbContext` hay Repository trực tiếp.
- **Business Logic** — điều phối use case, kiểm tra quyền theo trusted actor, áp rubric/settings.
  Không tham chiếu `AIVESDbContext`, không dùng kiểu HTTP hay View.
- **Data Access** — contract đọc/ghi, mapping entity, truy vấn EF Core. Repository và `DbContext` luôn
  **Scoped**; không bao giờ Singleton (tránh captive dependency và tranh chấp `ChangeTracker`).

---

## Thành phần chính

### Presentation — `StudentNameMVC`

| Thành phần | Vai trò |
| :--- | :--- |
| Razor Pages | `/Viva`, `/Account`, `/Admin/AiSettings`, `/ClassReports`, `/StudentReports` |
| SignalR Hub | `VivaHub` + `GradingGroup` — đẩy sự kiện phiên thi và trạng thái chấm điểm |
| Controllers | Các controller MVC còn lại đang chuyển dần sang Razor Pages |
| Client | `wwwroot/js` — MediaRecorder ghi âm, `speechSynthesis` đọc câu hỏi |

### Business Logic — `AIVES.BLL`

| Thành phần | Vai trò |
| :--- | :--- |
| Viva Service | `VivaSessionOrchestrator` — vòng đời phiên, đồng hồ, câu hỏi xoáy |
| Interfaces / DTOs | Contract giữa các tầng, model dùng chung |
| AI Providers | STT, TTS, LLM — adapter cho OpenAI / Gemini / OpenAI-compatible |
| Reporting | Tính báo cáo cá nhân, thống kê lớp |

### Data Access — `AIVES.DAL`

| Thành phần | Vai trò |
| :--- | :--- |
| Repositories | Account, Viva, Report — tất cả Scoped |
| Entity Models | `SystemAccount`, `VivaSession`, `VivaSessionTurn`, … |
| `AIVESDbContext` | EF Core, Database First, cấu hình constraint/index |

---

## Công nghệ

- **.NET 8** · ASP.NET Core Razor Pages + MVC
- **Entity Framework Core** — Database First
- **SQL Server** — kịch bản trong `database/`
- **SignalR** — realtime hai chiều
- **AI ngoài** — OpenAI / Gemini / OpenAI-compatible (HTTPS)
- **xUnit** — unit và integration test

---

## Chạy dự án

### 1. Tạo database

Chạy lần lượt các script trong `database/` trên SQL Server:

```
database/001-create-schema.sql
database/002-add-system-account-soft-delete.sql
database/003-create-viva-exam-core.sql
database/004-seed-viva-questions.sql
```

### 2. Cấu hình

Sửa `StudentNameMVC/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=AIVES;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "DefaultAdmin": {
    "Email": "admin@AIVESSystem.org",
    "Password": "@@abc123@@"
  }
}
```

API key AI **không** đặt trong `appsettings`. Cấu hình qua trang `/Admin/AiSettings` sau khi đăng nhập
Admin — key được mã hoá bằng Data Protection và lưu ngoài source.

### 3. Build và chạy

```bash
dotnet build AIVESSystem.sln
dotnet run --project StudentNameMVC/StudentNameMVC.csproj
```

### 4. Test

```bash
dotnet test AIVESSystem.sln
```

---

## Cấu trúc thư mục

```
.
├── AIVES.BLL/                # Business Logic Layer
│   ├── Interfaces/           # Contract giữa các tầng
│   ├── Models/               # DTO, model dùng chung
│   └── Services/             # Nghiệp vụ, orchestrator, AI adapter
├── AIVES.DAL/                # Data Access Layer
│   ├── Context/              # AIVESDbContext
│   ├── Entities/             # Entity EF Core
│   └── Repositories/         # Repository (Scoped)
├── AIVES.Tests/              # Unit + integration test
├── StudentNameMVC/           # Presentation Layer
│   ├── Pages/                # Razor Pages
│   ├── Hubs/                 # SignalR
│   └── wwwroot/              # Static assets, JS
├── database/                 # SQL script (001 → …)
├── docs/                     # Tài liệu, sơ đồ kiến trúc
└── AIVESSystem.sln
```

---

## Tài liệu

| File | Nội dung |
| :--- | :--- |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Ranh giới tầng, lifetime, quy ước persistence |
| [`docs/business-rules.md`](docs/business-rules.md) | Quy tắc nghiệp vụ đã chốt (DEC-01 … DEC-12) |
| [`docs/ASM2-CONTRACTS.md`](docs/ASM2-CONTRACTS.md) | Contract nhóm 3 — AI Interview Core |
| [`docs/database-connection-guide.md`](docs/database-connection-guide.md) | Hướng dẫn kết nối SQL Server |
| [`docs/architecture/aives-asm2.drawio`](docs/architecture/aives-asm2.drawio) | Sơ đồ kiến trúc (nguồn) |

---

## Ghi chú thiết kế

- **AI không tự chốt điểm.** Điểm AI chỉ là gợi ý; giảng viên xác nhận hoặc ghi đè.
- **Persistence first.** Trạng thái được ghi xuống SQL Server trước khi phát sự kiện SignalR, để client
  luôn nhận được dữ liệu đã bền vững.
- **Scope mỗi job.** Background worker tạo `IServiceScope` riêng cho từng job để resolve đúng scoped
  service.
- **Không lưu secret trong source.** API key nằm trong store mã hoá cục bộ, không commit.
