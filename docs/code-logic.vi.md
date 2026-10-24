# Logic code của demo-machine

Tài liệu ngắn mô tả luồng xử lý và cách các module trong `src/` ghép với nhau.
Tham chiếu chi tiết từng field/lệnh xem [spec-reference.md](spec-reference.md) và [cli-reference.md](cli-reference.md).

## Ý tưởng cốt lõi

Một file spec (`.demo.yaml` / `.json5` / `.toml`) là **nguồn sự thật duy nhất**. Toàn bộ phần còn lại là một pipeline thuần: spec → sự kiện → timeline → video. Mỗi giai đoạn ghi artifact ra đĩa, nên có thể dừng lại, kiểm tra, hoặc chạy lại từ giữa (`edit`, `analyze`) mà không phải quay lại từ đầu.

## Luồng chính (`demo-machine run`)

```text
spec file
  │  src/spec/loader.ts   → đọc yaml/json/json5/toml
  │  src/spec/schema.ts   → validate bằng zod, điền default
  ▼
DemoSpec
  │  src/validation/preflight.ts  → check tĩnh (selector, narration, pacing, xung đột action)
  │  src/runner/runner.ts         → spawn app (runner.command), đợi healthcheck/url
  │  src/capture/recorder.ts      → mở Playwright context có recordVideo + trace
  ▼
  │  src/playback/engine.ts       → chạy từng chapter/step
  │     ├─ overlays.ts  : inject con trỏ giả, focus ring, spotlight, redaction CSS
  │     ├─ cursor.ts    : SVG con trỏ + chuyển động bezier + ripple khi click
  │     ├─ actions.ts   : bảng actionHandlers → src/playback/handlers/*.ts
  │     ├─ narration-focus.ts : zoom/di chuyển camera tới phần tử đang được thuyết minh
  │     ├─ change-detection/  : xác nhận action thật sự làm UI đổi (dom, layout, style, aria)
  │     └─ guards + validation/monitors : console error, dialog, network, lifecycle
  ▼
ActionEvent[]  (events.json) + video.webm + trace.zip + screenshots/
  │  src/editor/timeline.ts   → dựng Timeline: intro, chapter title, content, callout, outro
  │  src/narration/*          → sinh script, gọi TTS, mix audio, xuất .srt/.vtt
  │  src/editor/renderers/ffmpeg.ts → filter graph (scale, zoom, drawtext, ghép audio)
  ▼
output.mp4
  │  src/quality/runner.ts → ~21 check: resolution, codec, fps, độ dài A/V,
  │                           thứ tự narration, screenshot, vị trí con trỏ, overlay lạ...
  ▼
quality.json (+ verification.json, metadata.json, share viewer tuỳ chọn)
```

Điều phối giai đoạn nằm ở `src/pipeline.ts` (logic thuần, dễ test) và `src/cli/pipeline.ts` (ghép I/O, ghi `output/latest.json`).

## Bản đồ module

| Thư mục           | Trách nhiệm                                                                     |
| ----------------- | ------------------------------------------------------------------------------- |
| `src/spec/`       | Load + schema zod cho spec, step, share config                                  |
| `src/validation/` | Check trước (preflight) / sau (postflight) capture, và các monitor gắn vào page |
| `src/runner/`     | Khởi động & dọn dẹp process app của người dùng                                  |
| `src/capture/`    | Playwright context, video/trace, ghi `events.json`, metadata                    |
| `src/playback/`   | Lái trình duyệt: action handler, overlay thị giác, change detection             |
| `src/redaction/`  | Blur selector nhạy cảm + quét secret trong DOM                                  |
| `src/editor/`     | Dựng `Timeline` và render (ffmpeg mặc định, remotion tuỳ chọn)                  |
| `src/narration/`  | Script, TTS provider (kokoro/openai/elevenlabs/piper), mix, phụ đề, auto-sync   |
| `src/quality/`    | Đo video đã render, chấm pass/fail, assurance report                            |
| `src/share/`      | Sinh trang xem tĩnh có chapter, poster, transcript                              |
| `src/mcp/`        | MCP server để agent gọi `validate` / `capture` / `run`                          |
| `src/cli/`        | Phân rã từng lệnh CLI; `src/cli.ts` chỉ là lớp commander mỏng                   |

## Ba cấu trúc dữ liệu xương sống

- **`DemoSpec`** (`src/spec/types.ts`) — spec sau khi zod điền default: `meta`, `runner`, `chapters[].steps[]`, cộng `pacing`, `visuals`, `narration`, `presentation`, `redaction`, `changeDetection`.
- **`ActionEvent`** (`src/playback/types.ts`, re-export qua `src/capture/types.ts`) — một bản ghi cho mỗi step đã chạy: `action`, `selector`, `timestamp`, `duration`, `boundingBox` của phần tử, `narration`, và `evidence.changeDetection` (tín hiệu chứng minh UI đã đổi). Đây là _bằng chứng_ để dựng timeline và chấm chất lượng; vị trí con trỏ thì nằm trong `screenshots/manifest.json` do `capture-hooks.ts` ghi.
- **`Timeline`** (`src/editor/types.ts`) — danh sách `Segment` có `startMs`/`endMs`, kiểu (`intro` | `chapter` | `content` | `callout` | `outro`), vùng `zoom` và `speedFactor` (đoạn chết > 3s bị tua nhanh 3×).

## Quy ước chi phối thiết kế

- **Phân tầng được `dependency-cruiser` ép cứng** (`.dependency-cruiser.cjs`): `playback` chỉ import được `spec`/`redaction`/`utils`, `capture` chỉ `spec`/`playback`/`utils`, `editor` chỉ `spec`/`capture`/`utils`. Hệ quả quan trọng: **`editor` không được chạm vào `playback`** — nó chỉ biết `ActionEvent` qua `src/capture/types.ts`, nhờ vậy render tách hẳn khỏi trình duyệt và `demo-machine edit events.json` chạy được mà không cần Playwright.
- **Action là registry, không phải switch** — thêm action mới = thêm handler trong `src/playback/handlers/` + một dòng trong map `actionHandlers` (`src/playback/actions.ts`) + schema trong `src/spec/step-schema.ts`.
- **Step `assert` phải không có hiệu ứng thị giác** — không `pulseFocus`, không `flashSpotlight`. Check `visual:assert-zero-effect` so sánh ảnh before/after để ép điều này.
- **Mọi giai đoạn đều ghi artifact** — vì thế `analyze` có thể dựng lại hồ sơ review từ một run cũ mà không cần quay hay render lại.
- **TTS mặc định chạy local** (kokoro) nên pipeline không phụ thuộc dịch vụ ngoài; provider khác chỉ là adapter trong `src/narration/providers/`.

## Điểm mở rộng thường dùng

| Muốn thêm              | Sửa ở                                                         |
| ---------------------- | ------------------------------------------------------------- |
| Một action mới         | `src/playback/handlers/` + `actions.ts` + `step-schema.ts`    |
| Một check chất lượng   | `src/quality/checks/` + đăng ký trong `src/quality/runner.ts` |
| Một giọng/provider TTS | `src/narration/providers/` + `provider.ts`                    |
| Một renderer           | `src/editor/renderers/` + `renderer.ts`                       |
| Một monitor lúc chạy   | `src/validation/monitors/` + `monitor-runner.ts`              |
| API công khai          | export lại trong `src/index.ts`                               |
