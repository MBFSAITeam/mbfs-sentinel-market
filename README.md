# mbfs-sentinel-market

Chợ plugin cho **MBFS Sentinel**. Repo này là nơi phân phối các AI plugin (dynamic pipeline) để
Sentinel GUI có thể tìm và cài đặt trực tiếp từ tab **Chợ** trong trang Plugins.

Repo public, không cần token — GUI tải qua `raw.githubusercontent.com`.

---

## Bố cục

```text
.
├── index.json                            # danh mục — file duy nhất GUI tải trước tiên
├── schema/index.schema.json              # JSON Schema của index.json
└── plugins/<plugin_name>/
    ├── manifest.toml                     # sinh tự động — copy từ dist/plugins/<name>/
    ├── windows-x86_64/<name>.dll         # sinh tự động
    ├── README.md                         # thủ công, tuỳ chọn — mô tả dài
    └── icon.png                          # thủ công, tuỳ chọn
```

`index.json` và mọi file trong `plugins/<name>/` **trừ** `README.md` / `icon.png` đều do script
sinh ra — đừng sửa tay, lần publish sau sẽ ghi đè.

Mỗi lần publish ghi đè tại chỗ (không tạo thư mục theo version). Lịch sử các bản cũ nằm trong git.

---

## `index.json`

```json
{
  "schema_version": 1,
  "generated_at": "2026-07-28T09:00:00Z",
  "plugins": [
    {
      "name": "weapon_detection",
      "display_name": "Nhận diện vũ khí",
      "description": "Phát hiện và cảnh báo hành vi sử dụng vũ khí",
      "version": "2.1.0",
      "abi_version": 12,
      "min_sentinel_version": "5.4.7",
      "license_feature": "weapon_detection",
      "capabilities": {
        "use_gpu": true, "use_batching": true,
        "use_tracking": true, "use_alert": true,
        "zones": ["detection_zone"]
      },
      "classes": ["Grenade", "Knife", "Pistol", "Rifle", "Sword"],
      "models": [{ "file": "mbfs_weapon_det_v6.onnx" }],
      "artifacts": {
        "windows-x86_64": {
          "library": "weapon_detection.dll",
          "manifest_url": "https://raw.githubusercontent.com/MBFSAITeam/mbfs-sentinel-market/main/plugins/weapon_detection/manifest.toml",
          "library_url": "https://raw.githubusercontent.com/MBFSAITeam/mbfs-sentinel-market/main/plugins/weapon_detection/windows-x86_64/weapon_detection.dll",
          "library_size": 1441280,
          "library_sha256": "…"
        }
      },
      "published_at": "2026-07-28T09:00:00Z"
    }
  ]
}
```

Vài điểm về schema đáng biết trước khi sửa nó:

- **URL trong `artifacts` là tuyệt đối.** Ngày nào DLL chuyển sang GitHub Release asset (khi lịch sử
  git phình vì blob nhị phân) hoặc sang nơi khác, chỉ publisher đổi — client không đổi dòng nào.
- **`models` là mảng object, không phải mảng string.** Hiện chỉ có `{"file": "..."}`; client dùng nó
  để kiểm tra model đã có sẵn trên máy chưa. Khi mở tính năng tải model, thêm `url` / `sha256` /
  `size` vào cùng object — không phải phá schema.
- **`artifacts` khoá theo nền tảng.** Hiện chỉ `windows-x86_64`; thêm `linux-x86_64`,
  `jetson-aarch64` sau mà không đổi schema.
- **`abi_version`** phải khớp `MBFS_SDK_ABI_VERSION` của bản Sentinel đang chạy, nếu không client
  từ chối cài (host sẽ không nạp được DLL).
- **`capabilities` copy nguyên khối `[capabilities]` của manifest** — đừng giả định toàn bool:
  `zones` là danh sách zone kind (`["detection_zone"]`).

---

## Cài đặt phía Sentinel

GUI đọc `index.json`, đối chiếu với `plugins/` trên máy, rồi tải `manifest.toml` + DLL vào
`<thư mục cài đặt>/plugins/<name>/`. **Cần khởi động lại Sentinel** thì plugin mới được đăng ký —
việc quét manifest chỉ chạy một lần lúc khởi động.

Plugin chỉ chạy được khi model ONNX nó khai báo đã có trong thư mục `models/`. Chợ **chưa** phân
phối model; thiếu model thì GUI hiện lý do và chặn cài.

### Dùng offline

Copy nguyên repo này ra USB/ổ mạng rồi trỏ `market.index_url` trong `config.yml` vào đó:

```yaml
market:
  index_url: "file:///D:/media/mbfs-sentinel-market/index.json"
```

Vì `artifacts` trong `index.json` là URL tuyệt đối, bản sinh cho media phải được sinh với đường dẫn
media đó (script publish trong repo `sentinel-core-ai` nhận tham số này) — khi ấy cả danh mục lẫn
DLL đều đọc từ đĩa, không cần mạng.
